---
doc_meta:
  id: TDD-notif-runtime-002
  title: Notification Runtime Persistence and Concurrency
  owner: Notification Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-005
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Notification Runtime Persistence and Concurrency

## Purpose

Define Notification PostgreSQL schema families, Delivery worker claim/attempt-start transaction boundaries, idempotency, inbox/outbox durability, RLS isolation, and crash behavior around external provider calls.

## Scope

Covers authoritative runtime persistence and concurrency patterns. Exact provider routing policy is TDD-004, Scheduling binding FSM is TDD-005, and callback security is TDD-007.

## Technical Context

PostgreSQL is the private authoritative Notification store. Provider delivery is asynchronous but does not require RabbitMQ/Kafka internally; ready Delivery rows are durable work. External provider I/O never occurs inside a database transaction.

## Component Design

```text
DeliveryWorker
  -> claim ready Delivery
  -> suppression evaluation
  -> resolve/freeze attempt realization
  -> atomically mark attempt STARTED
  -> provider I/O
  -> persist normalized outcome + outbox
```

The transaction that marks an attempt `STARTED` is the cancellation/attempt linearization point.

## Data Model

Core tables include:

- `notifications(notification_id, owner scope, lifecycle_state, communication_class, snapshot_hash, scheduled_at, created_at, cancelled_at)` where `cancelled_at` is the authoritative gate for not-started delivery
- `recipient_snapshots(snapshot_id, notification_id, endpoint_ciphertext_or_protected_value, endpoint_hash, bounded_metadata, immutable_hash)`
- `deliveries(delivery_id, notification_id, channel, state, ready_at, attempt_no, lease_until, terminal_at)`
- `delivery_attempts(attempt_id, delivery_id, attempt_no, state, provider_binding_id, routing_version, endpoint_identity, secret_ref_version, stable_delivery_identity, send_started_at, normalized_outcome, provider_message_id, evidence timestamps)`
- `notification_idempotency(application_id, tenant_scope, key, semantic_fingerprint, notification_id)`
- `notification_outbox(outbox_id, aggregate_id, event_type, payload, state, attempt_count, available_at, lease_until)`
- `provider_callback_inbox(provider_binding_id, provider_event_key, payload_hash, received_at, applied_at)`
- configuration/template tables defined contractually in TDD-003
- Scheduling registration/inbox tables defined in TDD-005

Critical constraints:

```sql
UNIQUE (application_id, tenant_scope, idempotency_key)
UNIQUE (delivery_id, attempt_no)
UNIQUE (provider_binding_id, provider_event_key)
```

Indexes cover `(state, ready_at)` for ready Deliveries and outbox `(state, available_at)`.

## API / Interface

Worker repository port exposes bounded lease/transition operations. Worker claim uses `FOR UPDATE SKIP LOCKED`, sets a short lease, and commits. Before external I/O, a second short transaction locks the Delivery, rechecks cancellation/suppression, creates/finalizes the attempt realization, sets attempt `STARTED` and `send_started_at`, then commits.

The provider call starts only after that commit.

## Algorithms / Logic

Ready claim query selects `READY` Deliveries with expired/no lease, ordered by `ready_at`, bounded per provider/Tenant bulkhead.

Attempt start transaction:

1. lock Delivery
2. ensure state remains `READY`
3. evaluate current suppression facts
4. if suppressed, persist `SUPPRESSED` and outbox, commit
5. resolve provider route/secret reference
6. insert immutable attempt evidence
7. transition Delivery to `ATTEMPTING`, set attempt `STARTED`/`send_started_at`
8. commit
9. perform external I/O

A crash after step 8 is treated conservatively as potentially ambiguous because the process boundary cannot prove whether the external call began. TDD-004 decides safe retry/reconciliation from provider capability.

## Configuration

- delivery claim batch default 100
- worker lease 60 s
- DB statement timeout 5 s
- max ready scan batch 1000
- outbox payload max 64 KiB
- callback payload max defined in TDD-007

## Security Notes

Runtime roles are least-privilege DML roles; migration role is separate. Tenant-scoped state uses enterprise RLS where applicable. Recipient endpoints are protected at rest according to data classification and masked in query projections.

Raw provider secret values never enter tables.

## Failure Handling

- crash before acceptance commit: no Notification exists
- crash after acceptance commit: Notification/Delivery/outbox survive
- crash before attempt STARTED commit: cancellation or another worker can safely decide next action after lease
- crash after STARTED commit: outcome may be `UNKNOWN`; never assume failure
- provider outage does not lose Delivery
- DB outage pauses new attempt starts; no provider send starts without durable start evidence

## Observability

Metrics cover ready backlog, claim latency, lease expiry, attempt starts, DB errors, outbox age, and RLS denial. Traces separate claim/start/provider/result spans.

## Performance Notes

No external I/O is held under row locks. Partial indexes keep ready/outbox scans bounded. Channel/provider bulkheads prevent one provider from monopolizing workers. Partitioning is deferred until measured retention/cardinality requires it.

## Testing Strategy

Production-equivalent PostgreSQL tests cover concurrent workers, cancel-vs-start race, process kill at every boundary, lease recovery, RLS isolation, unique callback/idempotency constraints, outbox atomicity, provider-outage backlog, and 10x forecast acceptance/worker load.

## Operational Notes

Retention/vacuum/index health and ready/outbox backlog are monitored. Direct SQL repair is not a normal support path. Backups include accepted Notification, Delivery, attempt, idempotency, outbox, and Scheduling registration state.

## Traceability

Implements SAD-005 State & Data Architecture and Channel Dispatch Worker concurrency. Conforms to PAD-PLT-005 RPO/cancellation/attempt evidence rules. Provider ambiguity policy is TDD-004.
