---
doc_meta:
  id: TDD-notif-runtime-007
  title: Provider Callback and Receipt Normalization
  owner: Notification Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-005
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Provider Callback and Receipt Normalization

## Purpose

Define authenticated provider callback ingestion, replay/dedup protection, out-of-order receipt handling, normalized Delivery transitions, `UNKNOWN` resolution, and durable acknowledgement.

## Scope

Covers inbound provider callbacks/receipts only. Outbound provider send policy is TDD-004 and governed webhook egress is TDD-008.

## Technical Context

Providers deliver duplicate and out-of-order callbacks and use different signature formats/status vocabularies. Notification must authenticate at the adapter, normalize semantics, commit state before acknowledgement, and never let a late weaker event regress proven state.

## Component Design

```text
Provider Callback Route
  -> Provider-specific Authenticator
  -> Callback Normalizer
  -> Callback Inbox/Dedup
  -> Delivery Transition Policy
  -> local outbox
```

Raw provider payload models do not enter domain packages.

## Data Model

Callback inbox key is `(provider_binding_id, provider_event_key)`. Prefer provider event ID; when unavailable, derive a stable SHA-256 fingerprint from authenticated immutable callback fields.

Persist safe evidence:
- provider event ID/fingerprint
- provider message ID
- received timestamp
- authenticated signature scheme/version
- raw payload hash
- normalized event type/status
- apply outcome
- correlation to Attempt/Delivery

Raw payload retention is governed separately and defaults to minimal required evidence.

## API / Interface

Provider-specific callback routes are isolated by adapter/binding. Maximum request body is 256 KiB.

Authenticator contract validates exact raw body, provider timestamp/nonce where supported, signature/key reference, and configured clock-skew window before JSON semantic parsing.

HTTP success is returned only after inbox dedup and any applicable Delivery mutation commit.

## Algorithms / Logic

1. read bounded raw body
2. authenticate signature before trusting fields
3. derive provider event key
4. insert/lock inbox dedup row
5. resolve Attempt/Delivery by provider identifiers and binding
6. normalize provider status
7. apply monotonic transition policy
8. write lifecycle outbox
9. commit
10. acknowledge provider

Monotonic examples:
- `PROVIDER_ACCEPTED -> DELIVERED` allowed
- `UNKNOWN -> DELIVERED` allowed when callback proves it
- `UNKNOWN -> FAILED_PERMANENT` allowed only when callback proves no delivery/terminal rejection
- `DELIVERED -> PROVIDER_ACCEPTED` ignored as stale
- duplicate callback is idempotent

## Configuration

- callback body max 256 KiB
- provider timestamp skew default 5 min where supported
- per-binding callback rate limit
- signature key rotation overlap window
- unresolved callback correlation retention/evidence window

## Security Notes

Signature verification uses constant-time comparison where applicable and exact raw bytes required by provider contracts. Replay protection uses timestamp/nonce/event dedup. Unknown binding/provider message IDs do not disclose internal IDs in responses.

Secrets/signature keys come from secret references and are excluded from logs.

## Failure Handling

Authentication failure returns 401/403 without state mutation. Valid callback with transient DB failure returns retryable server error so provider can retry. Unknown correlation is durably recorded/parked when policy requires investigation rather than silently discarded.

## Observability

Metrics:

- `notification_callback_total{provider,outcome}`
- `notification_callback_auth_failures_total`
- `notification_callback_duplicates_total`
- `notification_callback_unmatched_total`
- `notification_callback_state_advances_total{from,to}`

Trace spans contain provider/binding and internal IDs but no raw body/signature.

## Performance Notes

Callback transaction is short and indexed by provider event/message ID. Heavy reconciliation never runs synchronously in the provider request.

## Testing Strategy

Provider contract tests cover valid/invalid signature, key rotation, replay, duplicate callback, out-of-order receipts, unknown correlation, `UNKNOWN` resolution, final-state non-regression, DB failure before ack, payload limits, and telemetry redaction.

## Operational Notes

Unmatched/authenticated callbacks have bounded parking and reconciliation workflow. Operators see normalized state and provider evidence without needing provider-console access for ordinary incidents.

## Traceability

Implements SAD-005 Callback/Receipt Normalization and Provider Callback flow. Complements provider outcome policy TDD-004 and lifecycle publication TDD-006.
