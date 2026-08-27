---
doc_meta:
  id: TDD-notif-experience-002
  title: Notification Operations User Interface
  owner: Notification Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-015
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Notification Operations User Interface

## Purpose

Define React feature boundaries and operator workflows for templates, channel/sender profiles, Application Notification Profiles, Provider Bindings, test-send, Delivery history, `UNKNOWN` reconciliation, quota/health, and audit correlation.

## Scope

Covers presentation and interaction only. Notification Runtime remains authoritative for configuration, delivery, retry, suppression, reconciliation, and secrets.

## Technical Context

The UI consumes only same-origin BFF routes and generated/validated Notification contracts. It presents provider-independent semantics and does not become a provider-console clone.

## Component Design

Feature modules:

- `template-management`
- `channel-profile-management`
- `application-notification-profile-management`
- `provider-binding`
- `notification-delivery-explorer`
- `reconciliation-operations`
- `test-send`
- `quota-usage`
- `provider-health`
- `audit-correlation`

Shared state is limited to session/context, routing, and bounded query cache.

## Data Model

Client models come from Notification contracts. Secret fields are write-only form values and never enter persisted query caches. Delivery timeline distinguishes:

`READY`, `ATTEMPTING`, `PROVIDER_ACCEPTED`, `UNKNOWN`, `DELIVERED`, `FAILED_PERMANENT`, `SUPPRESSED`, `CANCELLED`.

Recipient endpoints are masked according to operator permission.

## API / Interface

All feature requests use `/api/notification/*`.

Template publication and provider/profile mutation show current Tenant/Application scope and require server-provided version/ETag. Test-send requires explicit scope, recipient/channel preview, reason, and privileged confirmation.

`UNKNOWN` reconciliation screen shows immutable attempt evidence, provider capabilities, allowed safe actions, and reason requirement. It never offers blind failover when Runtime policy forbids it.

## Algorithms / Logic

Context switch destroys old-generation cache, forms, selected deliveries, and pending mutations.

Mutation conflicts fetch current server state; no last-write-wins UI behavior.

Provider acceptance is visually distinct from final delivery. Color is never the only status signal.

Secret entry is intentionally lost on reload/navigation/context switch.

## Configuration

- default history page 50, max 200
- bounded timeline window
- test-send form rate limited by Runtime and client duplicate-submit guard
- accessibility target WCAG 2.2 AA
- rendered preview size follows BFF limit

## Security Notes

UI visibility is not authorization. Secret values are never re-rendered after submission. Template preview always uses the sandbox boundary. Raw provider errors/recipient PII are masked/normalized by contract.

## Failure Handling

Partial provider-health failure does not break configuration/history screens. Ambiguous test-send response retains its idempotency identity for explicit recovery. `UNKNOWN` actions are disabled unless Runtime returns an allowed action set.

## Observability

Browser telemetry records feature/outcome/performance/correlation only. No secrets, raw recipient endpoints, full rendered content, access tokens, or callback payloads are captured.

## Performance Notes

History uses server pagination and windowed rendering. Template editor/preview work is bounded. Provider-health polling uses backoff and pauses when page is hidden.

## Testing Strategy

E2E tests cover template create/publish/preview, profile/provider binding, write-only secret, test-send abuse/duplicate submit, Delivery state rendering, provider-accepted vs delivered, `UNKNOWN` reconciliation safety, context switch, stale ETag, PII masking, accessibility, keyboard/focus behavior, and partial API degradation.

## Operational Notes

UI rollback is independent from Runtime. Provider replacement, broker-profile change, or Scheduling internals require no UI rewrite while Notification contracts remain compatible.

## Traceability

Implements SAD-015 Template Management, Channel/Profile administration, Delivery Explorer, Reconciliation, Quota/Health, test-send, and Audit Correlation. Browser/security boundary is TDD-notif-experience-001.
