---
doc_meta:
  id: TDD-notif-runtime-006
  title: Notification Messaging Outbox and Lifecycle Contracts
  owner: Notification Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-005
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Notification Messaging Outbox and Lifecycle Contracts

## Purpose

Define transport-neutral Notification lifecycle publication, source-local transactional outbox, Direct/RabbitMQ/Kafka messaging adapters, inbound durable acceptance rules, and schema compatibility without coupling provider workers to a broker.

## Scope

Covers Notification lifecycle events and asynchronous Notification/deferred-command integration. Provider delivery workers remain PostgreSQL-backed internal execution. Scheduling-specific trigger semantics are TDD-005.

## Technical Context

RabbitMQ is the default queue profile, Kafka is the supported retained stream profile, and Direct durable acceptance is allowed for bounded relationships. One logical message contract has one primary path per environment.

## Component Design

```text
Domain Tx -> notification_outbox
OutboxRelay -> MessagingPort -> Direct | RabbitMQ | Kafka

Inbound Adapter -> durable Inbox/Notification acceptance -> ACK
```

Broker SDK types stay in adapters.

## Data Model

Outbox state is `PENDING`, `IN_FLIGHT`, `ACCEPTED` with bounded lease/backoff. Lifecycle CloudEvent families:

- `com.scnehaux.notification.accepted.v1`
- `com.scnehaux.notification.cancelled.v1`
- `com.scnehaux.notification.delivery.provider_accepted.v1`
- `com.scnehaux.notification.delivery.delivered.v1`
- `com.scnehaux.notification.delivery.failed.v1`
- `com.scnehaux.notification.delivery.unknown.v1`

Event data contains IDs, ownership/correlation, channel, normalized state, safe reason/evidence references. It excludes raw recipient endpoints, rendered content, and provider credentials by default.

## API / Interface

```go
type MessagingPort interface {
    Publish(ctx context.Context, event CloudEvent) (DurableAcceptance, error)
}
```

RabbitMQ production uses durable exchange/queue, persistent messages, quorum queues where C1 applies, mandatory routing, publisher confirms, and consumer ACK only after local durable processing.

Kafka uses replicated topic, compatible schema, explicit key, and consumer commit after local durability.

Direct returns success only after target durable deduplication/acceptance.

## Algorithms / Logic

Outbox relay follows the same crash-safe lease/publish/mark pattern as Scheduling but uses Notification-owned tables and lifecycle contracts.

Inbound asynchronous Notification intent requires a source command identity; duplicate delivery resolves to the same accepted Notification through idempotency. Consumer ACK/offset advancement occurs only after local authoritative commit.

## Configuration

- primary messaging profile: `rabbitmq`, `kafka`, or `direct`
- relay batch default 200
- lease 30 s
- payload limit 64 KiB
- event schema compatibility mode: backward-compatible within v1 family

Startup rejects mutually exclusive primary adapters for the same logical contract.

## Security Notes

Events minimize PII and never contain secrets. Broker credentials are secret references resolved at runtime. Tenant/application ownership is carried for authorization/correlation but is not accepted as proof when a receiving protected API can derive stronger trust context.

## Failure Handling

Messaging outage leaves committed outbox rows pending. Unroutable/poison events park and alert without blocking unrelated lifecycle events. Ambiguous direct acceptance is reconciled before blind retry unless target contract guarantees idempotency.

## Observability

Metrics cover outbox lag, publish outcome/profile, inbound duplicate count, parked messages, and contract-validation failures. Trace context propagates through CloudEvents.

## Performance Notes

Provider worker throughput is not coupled to lifecycle broker throughput. Outbox relay concurrency and payload size are bounded. Large rendered content is referenced/queried through owned APIs, not copied into events.

## Testing Strategy

Common profile suite covers duplicate publication, process kill, broker outage, poison/unroutable event, schema compatibility, ACK-after-commit, inbound idempotency, privacy payload checks, and rejection of dual primary adapters.

## Operational Notes

Profile migration uses drain/reconciliation and contract-compatible consumers; no blind RabbitMQ+Kafka dual-publish. Broker admin UI is not Notification product UX.

## Traceability

Implements SAD-005 Outbox Relay and Messaging Port/Adapters. Conforms to the transactional-publication and durable-messaging architecture referenced by SAD-005. Scheduling wake-up composition remains TDD-005.
