---
doc_meta:
  id: TDD-notif-runtime-001
  title: Notification Lifecycle and Frozen Snapshot
  owner: Notification Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-005
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Notification Lifecycle and Frozen Snapshot

## Purpose

Define Notification and Delivery lifecycle semantics, immutable communication snapshot, cancellation gate, dynamic suppression boundary, idempotent acceptance, and the distinction between provider acceptance and final delivery.

## Scope

Covers Notification aggregate behavior from accepted intent through Delivery planning, scheduling readiness, cancellation, and aggregate completion. Provider call mechanics, persistence DDL, callback normalization, and Scheduling registration are covered by sibling TDDs.

## Technical Context

Products own why communication is needed and business-recipient eligibility. Notification owns how accepted communication is rendered and delivered. Frozen communication semantics must remain immutable after acceptance, while operational provider realization is late-bound by default.

Current channel/legal/platform suppression is evaluated before a not-yet-started provider attempt and is not frozen into the acceptance snapshot.

## Component Design

```text
NotificationIngress
  -> NotificationCommandService
      -> Notification Aggregate
      -> SnapshotFactory
      -> DeliveryPlanner
```

Notification lifecycle states are `ACCEPTED`, `SCHEDULE_PENDING`, `SCHEDULED`, `ACTIVE`, and `COMPLETED`; cancellation is represented by an authoritative `cancelled_at` delivery gate rather than by rewriting already-started Delivery outcomes.

Delivery states: `PLANNED`, `READY`, `ATTEMPTING`, `PROVIDER_ACCEPTED`, `DELIVERED`, `FAILED_PERMANENT`, `UNKNOWN`, `SUPPRESSED`, `CANCELLED`.

Terminal Delivery states are `DELIVERED`, `FAILED_PERMANENT`, `SUPPRESSED`, and `CANCELLED`. `UNKNOWN` is non-terminal but blocks blind retry/failover. Cancellation changes only not-started Deliveries to `CANCELLED`; an already-started Delivery retains its real external outcome and may later become `PROVIDER_ACCEPTED`, `DELIVERED`, `FAILED_PERMANENT`, or remain `UNKNOWN`.

## Data Model

Frozen Notification semantics include:

- recipient snapshot endpoint and bounded recipient metadata
- immutable template family/version/channel variant
- validated template data
- selected channel
- logical sender identity when required
- immutable attachment version references
- communication class
- business correlation

These fields receive immutable version/hash evidence. Provider binding, endpoint, secret reference version, failover route, and rate-limit state are not part of the frozen Notification unless an explicit governed pin is requested.

Aggregate status is derived from scheduling state, `cancelled_at`, and Delivery states; cancellation never rewrites immutable started-attempt or Delivery outcome history.

## API / Interface

Domain commands:

- `AcceptNotification`
- `MarkSchedulePending`
- `BindSchedule`
- `ActivateScheduledDelivery`
- `CancelNotification`
- `MarkDeliveryReady`
- `ApplySuppressionDecision`
- provider-attempt transitions delegated to TDD-004

Every acceptance has a stable scoped idempotency key. Equivalent retry returns the same `notification_id`; conflicting reuse is rejected.

## Algorithms / Logic

Acceptance:

1. authorize scope and communication class
2. validate template data against immutable schema
3. snapshot recipient/content/attachment semantics
4. plan one or more bounded Deliveries
5. persist Notification, snapshots, Deliveries, idempotency, and outbox atomically
6. if future delivery, persist schedule-registration intent in the same local transaction

Cancellation linearization:
- if cancellation commits before a Delivery transitions to provider-attempt `STARTED`, no external attempt may begin
- if the attempt-start transaction committed, cancellation blocks any later attempt but leaves that started Delivery's outcome state intact and does not claim external retraction
- late Scheduling triggers for a cancelled Notification are durable no-ops

Suppression is checked immediately before attempt start for a not-yet-started Delivery and records policy evidence.

## Configuration

- maximum recipients per accepted Notification plan: deployment-governed bounded value
- maximum template data: 64 KiB per Delivery
- maximum attachment references: 20
- idempotency key maximum: 200 characters
- snapshot hashing: SHA-256

Fan-out beyond configured per-command bounds is expanded through asynchronous bounded work.

## Security Notes

Recipient endpoints are PII and are redacted/masked in logs, traces, and general operator views. Template data cannot execute arbitrary Product code. Raw provider credentials are never stored in Notification state.

Business-specific consent remains upstream; Notification suppression only applies within the declared communication class/policy authority.

## Failure Handling

A failed acceptance transaction produces no partial Notification. Invalid template/schema/recipient input fails before persistence. Scheduling registration failure after acceptance does not lose the Notification; it remains recoverable in `SCHEDULE_PENDING`.

Cancellation races follow the attempt-start linearization and never fabricate provider retraction.

## Observability

Metrics:

- `notification_accept_total{channel,outcome}`
- `notification_delivery_state_total{state}`
- `notification_suppressed_total{reason}`
- `notification_cancel_total`
- `notification_snapshot_validation_errors_total`

Trace spans avoid raw recipient/template data.

## Performance Notes

Acceptance performs no provider network I/O and no Scheduling transaction. Snapshot validation is bounded. Large fan-out is never one unbounded DB transaction.

## Testing Strategy

Tests cover duplicate acceptance, conflicting idempotency, immutable snapshot enforcement, dynamic suppression after acceptance, cancellation before/after attempt start, scheduled activation, late duplicate occurrence no-op, provider acceptance distinct from delivery, fan-out bounds, PII redaction, and terminal aggregate derivation.

## Operational Notes

Operators can distinguish accepted, scheduled, active, provider-accepted, delivered, failed, unknown, suppressed, and cancelled states. Historical snapshots and Delivery evidence are immutable.

## Traceability

Implements SAD-005 Notification Aggregate, Recipient Snapshot, Delivery Planning, suppression, and cancellation semantics; conforms to PAD-PLT-005 domain policies and STD-GLB-010 scheduled-communication rules. Persistence is TDD-002; provider attempts are TDD-004; Scheduling binding is TDD-005.
