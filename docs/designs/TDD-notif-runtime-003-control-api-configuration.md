---
doc_meta:
  id: TDD-notif-runtime-003
  title: Notification Control API and Configuration Model
  owner: Notification Platform Team
  version: 1.1.0
  status: approved
  classification: restricted
  parent_sad: SAD-005
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-28
---
# Notification Control API and Configuration Model

## Purpose

Define versioned Notification command/query APIs and administration contracts for Template Family/Version/Channel Variant, Channel/Sender Profile, Application Notification Profile, Provider Binding, test-send, and secret-reference workflows.

## Scope

Covers synchronous control-plane APIs and configuration ownership. Provider delivery execution, callbacks, webhook egress, and Scheduling composition are sibling TDDs.

## Technical Context

Notification owns communication configuration and profile mapping; it does not own Tenant/application identity or secret custody. Browser use is mediated by Notification Experience BFF. Products may use the Runtime API server-to-server.

## Component Design

Services:

- `NotificationCommandService`
- `TemplateService`
- `ChannelProfileService`
- `ApplicationNotificationProfileService`
- `ProviderBindingService`
- `TestSendService`
- `OperationsQueryService`

Configuration mutations emit outbox lifecycle/evidence facts.

## Data Model

Configuration entities are versioned:

- Template Family: stable key and lifecycle
- Template Version: immutable content/data-schema version
- Channel Variant: channel-specific immutable realization
- Channel/Sender Profile: logical sender identity and channel policy
- Provider Binding: provider adapter, non-secret endpoint/config, `secret_ref`
- Application Notification Profile: `(application_id, tenant_scope, channel, communication_class)` -> approved template/profile/routing policy

Provider secret fields never appear in read models. Only secret-reference metadata such as version/updated timestamp may be returned.

## API / Interface

Base `/v1`.

Notification:
- `POST /notifications` with `Idempotency-Key`
- `GET /notifications/{id}`
- `POST /notifications/{id}:cancel`
- Delivery/history queries

Templates:
- CRUD family metadata
- `POST /template-families/{id}/versions`
- publication/disable endpoints
- preview/validate endpoints

Profiles:
- Channel/Sender Profile CRUD
- Application Notification Profile CRUD
- Provider Binding CRUD

Secrets:
- `POST /provider-bindings/{id}:credentials` accepts write-only credential material and returns reference metadata only

Operations:
- `POST /test-sends`
- reconciliation/retry endpoints

Errors use RFC 9457. Mutating configuration uses ETag/`If-Match`. Privileged operations require reason.

## Algorithms / Logic

Notification acceptance computes a canonical semantic fingerprint similar to Scheduling idempotency and returns the same Notification on equivalent retry.

Template Version publication:
1. validate immutable data schema
2. validate channel variant syntax/safety
3. persist immutable version
4. emit publication evidence
5. never mutate published content in place; publish a new version instead

Application Notification Profile resolution uses validated local application/Tenant ownership context and current enabled configuration. Normal delivery does not synchronously query Organization.

## Configuration

API limits:
- request body 1 MiB
- template content 512 KiB
- template schema 128 KiB
- list page 50 default, 200 max
- test-send strict per-user/application rate limit
- credential-registration body excluded from generic retry middleware and body logging

## Security Notes

Credential registration is a write-only privileged path. Request bodies containing secrets are never logged, traced, cached, queued, or replayed by the Experience. Each credential write carries a stable server-generated `credential_operation_id`. Runtime sends it to the secret boundary, which treats equivalent retries as the same logical write/rotation result. Runtime persists only `secret_ref`, secret metadata/version, and `credential_operation_id`.

Template preview is treated as untrusted content; Experience sandbox rules are in TDD-notif-experience-001.

## Failure Handling

Configuration writes are atomic locally. Secret write followed by local metadata failure is reconciled by `credential_operation_id`: Runtime retries/queries the same secret operation and repairs only missing local reference metadata. It never creates an unrelated second credential version because a local commit/result was lost. Test-send cannot bypass normal provider authorization, suppression, rate limiting, or audit.

## Observability

Metrics cover API latency/outcome, template validation, profile resolution failure, secret-registration outcome without secret content, test-send attempts, and configuration conflicts. Audit facts include template publication, profile/provider mutation, credential rotation metadata, and test-send reason/scope.

## Performance Notes

Profile resolution uses indexed local state and bounded cache only as non-authoritative acceleration. Published template versions are immutable and cache-friendly. Admin/list traffic is isolated from delivery worker capacity.

## Testing Strategy

Contract/security tests cover immutable template versions, schema validation, ETag conflict, application/Tenant spoofing, secret non-return/non-log, profile resolution, disabled binding, test-send abuse/rate limit, idempotent Notification acceptance, and OpenAPI backward compatibility.

## Operational Notes

OpenAPI contracts live under `packages/contracts/openapi`. Configuration rollback selects a previously approved version/profile; it never rewrites historical attempt evidence.

## Traceability

Implements SAD-005 Notification Ingress, Template & Data Schema, Channel/Sender Profile, Application Notification Profile, Provider Binding, and admin query surfaces. Conforms to PAD-PLT-005 ownership and secret-boundary policies.
