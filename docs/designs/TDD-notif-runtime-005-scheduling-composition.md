---
doc_meta:
  id: TDD-notif-runtime-005
  title: Notification and Scheduling Composition
  owner: Notification Platform Team
  version: 1.1.0
  status: approved
  classification: restricted
  parent_sad: SAD-005
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-28
---
# Notification and Scheduling Composition

## Purpose

Define the durable cross-platform composition between Notification and Scheduling for Frozen Notifications and Deferred Notification Commands without a distributed transaction, including stable registration identity, binding reconciliation, occurrence deduplication, and cancellation races.

## Scope

Covers Notification-owned schedule registration state, Scheduling Control API calls, binding recovery, Scheduling trigger acceptance, deferred command validation, and asynchronous Schedule cancellation. Generic Scheduler internals remain outside this TDD.

## Technical Context

Frozen Notification is accepted and snapshotted before Schedule registration. Notification must survive a Scheduling timeout or lost response without becoming silently unscheduled or creating duplicate Schedules. Scheduling trigger delivery is at-least-once.

When current business truth is required at due time, this composition is not used to bypass the owning Product; Scheduling targets the Product Worker first.

## Component Design

```text
Notification Acceptance Tx
  -> schedule_registration_intent

RegistrationWorker
  -> Scheduling API
  -> BindingRepository

OccurrenceConsumer
  -> occurrence_inbox
  -> Delivery activation

CancellationWorker
  -> Scheduling cancel
```

## Data Model

`notification_schedule_registrations`:

- `notification_id`
- `generation` starting at 1
- `state`: `PENDING`, `BOUND`, `CANCEL_PENDING`, `CANCELLED`, `ERROR`
- `idempotency_key`
- `schedule_id` nullable until recovered
- `schedule_version` nullable until bound/recovered
- `last_attempt_at`, `next_attempt_at`, `attempt_count`
- request semantic hash
- reconciliation metadata

Unique `(notification_id, generation)` and unique `idempotency_key`.

Idempotency key format is stable logical identity `notif-schedule:<notification_id>:<generation>`; the concrete HTTP header value may be hashed/encoded.

`notification_schedule_occurrence_inbox` has unique `occurrence_id`, `schedule_id`, `notification_id`, receive/applied timestamps.

### Scheduling Binding State Machine

| Current | Event | Next |
| --- | --- | --- |
| none | Frozen Notification accepted | `PENDING` |
| `PENDING` | create/recovery succeeds | `BOUND` |
| `PENDING` | transient/ambiguous Scheduling result | `PENDING` with retry metadata |
| `BOUND` | Notification cancelled | `CANCEL_PENDING` |
| `CANCEL_PENDING` | cancel succeeds/Schedule terminal | `CANCELLED` |
| `CANCEL_PENDING` | stale Schedule version | remain, refresh owned version, retry |
| any non-terminal | unrecoverable ownership/contract mismatch | `ERROR` |

`ERROR` never creates a replacement Schedule with a new registration generation unless a new business intent explicitly requires it.

## API / Interface

Frozen registration sends a one-time Schedule request containing:
- target registered Notification wake-up contract
- original `scheduled_at`
- bounded trigger `{notification_id, generation}`
- stable `Idempotency-Key`

Trigger acceptance consumes `com.scnehaux.scheduling.occurrence.due.v1`.

Deferred Notification Command target uses a separately registered contract and bounded immutable input sufficient for Notification creation. It cannot carry provider credentials or unbounded recipient/content datasets.

## Algorithms / Logic

Registration:

1. acceptance transaction stores frozen Notification + `PENDING` registration
2. worker calls Scheduling create with stable key
3. success persists `schedule_id`, returned `schedule_version`, and `BOUND`
4. timeout/ambiguous response retries the same key
5. equivalent retry receives the same Schedule
6. if local binding persistence is lost after Scheduler success, the next retry/recovery returns the same Schedule and repairs the binding

Occurrence consume:

1. authenticate/validate registered Scheduling contract
2. insert inbox by `occurrence_id`
3. lock Notification
4. if terminally cancelled, mark inbox applied no-op
5. verify bound Schedule/generation
6. transition eligible Delivery to `READY`
7. commit before transport acknowledgement

Cancellation:
1. Notification cancellation commits locally first
2. registration becomes `CANCEL_PENDING`
3. worker attempts Scheduler cancel asynchronously with the last bound `schedule_version`
4. on `412` stale version, worker reads the owned Schedule, confirms it is still cancellable, updates the bound version, and retries
5. late/duplicate Occurrence remains a no-op against cancelled Notification

### Lock and Acknowledgement Contract

Occurrence consumption locks inbox identity before Notification, commits inbox plus activation before broker acknowledgement, and deduplicates on the same `occurrence_id`. Scheduling API calls and broker acknowledgements occur outside Notification DB row locks. No cross-platform call participates in a Notification transaction.

## Configuration

- registration retry starts 500 ms with jitter, max 30 s
- reconciliation age alert 60 s
- occurrence inbox retention >= Notification evidence retention minimum
- Scheduling client timeout 5 s
- bounded deferred-command payload <= 32 KiB

## Security Notes

Scheduling payload carries no provider secret. Notification validates application/Tenant ownership on deferred commands. A Schedule target cannot override Application Notification Profile/provider configuration.

Trigger authenticity follows the selected Scheduling delivery profile and registered contract.

## Failure Handling

Every process-loss point is recoverable:
- after Notification commit before Scheduler call: pending row remains
- after Scheduler commit before response: retry same key
- after response before binding commit: retry/recover same Schedule
- after occurrence inbox commit before ack: duplicate delivery dedupes
- Scheduler cancellation failure: local terminal Notification remains authoritative

## Observability

Metrics:

- `notification_schedule_registration_total{outcome}`
- `notification_schedule_registration_age_seconds`
- `notification_schedule_binding_reconciled_total`
- `notification_schedule_occurrence_total{outcome}`
- `notification_schedule_late_cancelled_noop_total`

Traces propagate Notification/Schedule/Occurrence correlation without content/recipient data.

## Performance Notes

Scheduling is never called inside the Notification acceptance transaction. Registration and cancellation use independent bounded workers. Occurrence acceptance is a short local transaction before ack.

## Testing Strategy

Fault-injection tests kill processes between every registration step, verify same Schedule ID on retry, test conflicting generation protection, duplicate occurrence, cancelled late trigger, Scheduler outage, binding reconciliation, direct/RabbitMQ/Kafka trigger profiles, and deferred-command size/secret rejection.

## Operational Notes

Operations surface `PENDING`/ambiguous bindings and reconciliation actions. Operators do not create replacement Schedules with new idempotency keys as an incident shortcut.

## Traceability

Implements SAD-005 Scheduling Registration/Binding/Reconciliation and Trigger Acceptance modules. Conforms to PAD-PLT-005 scheduled-notification policies and STD-GLB-010 §3.1 scheduled communication modes.
