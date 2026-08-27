---
doc_meta:
  id: TDD-notif-runtime-004
  title: Delivery Attempt and Provider Outcome Policy
  owner: Notification Platform Team
  version: 1.1.0
  status: approved
  classification: restricted
  parent_sad: SAD-005
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-28
---
# Delivery Attempt and Provider Outcome Policy

## Purpose

Define Delivery worker execution, provider capability model, routing/failover, immutable per-attempt realization, retry budgets, circuit breaking, suppression handoff, and first-class `UNKNOWN` outcome handling.

## Scope

Covers external communication providers for Email, WhatsApp/messaging, SMS, Push, and provider-style delivery adapters. Governed Webhook has additional network-security rules in TDD-008. Callback normalization is TDD-007.

## Technical Context

Provider semantics differ materially. Some support stable idempotency, status lookup, callback receipts, retraction, or final-delivery evidence; others do not. The runtime must not pretend those capabilities are uniform.

Provider acceptance is not final delivery. An ambiguous external side effect is `UNKNOWN`, not transient failure.

## Component Design

```text
DeliveryWorker
  -> ProviderPolicy
  -> RoutingPolicy
  -> ProviderCapabilityRegistry
  -> ProviderAdapter
  -> ReconciliationService
```

Provider SDK/model types terminate inside adapters. Domain code sees normalized outcomes.

## Data Model

Capability record per provider/channel binding:

| Capability | Values |
| --- | --- |
| idempotency | `NONE`, `STABLE_KEY` |
| reconciliation | `NONE`, `BY_DELIVERY_ID`, `BY_PROVIDER_ID` |
| callback | `NONE`, `AUTHENTICATED` |
| final_receipt | boolean |
| retraction | `NONE`, `SUPPORTED` |
| retry_after | boolean |

Each Attempt freezes:
- provider identity
- binding/routing policy version
- endpoint identity
- secret reference/version metadata
- stable delivery identity
- attempt number
- request semantic hash
- send-start timestamp

Raw secret values are excluded.

## API / Interface

```go
type ProviderAdapter interface {
    Send(ctx context.Context, req ProviderSendRequest) ProviderSendResult
    Lookup(ctx context.Context, ref ProviderLookupRef) ProviderLookupResult
    Retract(ctx context.Context, ref ProviderMessageRef) ProviderRetractResult
}
```

Unsupported capability methods return a typed `CapabilityUnsupported`, not fabricated success.

### Provider Safety Classification

Unknown provider capabilities default conservatively to no idempotency, no reconciliation, no retraction, and no final-receipt claim.

| Observation | Capability | Decision |
| --- | --- | --- |
| explicit local/pre-send failure | any | bounded retry |
| permanent rejection proving no effect | any | permanent failure |
| timeout after request may have left process | stable idempotency | retry same stable identity or reconcile |
| timeout after request may have left process | no idempotency, lookup exists | `UNKNOWN`, reconcile |
| timeout after request may have left process | no idempotency, no lookup | `UNKNOWN`, park for evidence-based resolution |
| prior effect proven absent | any | new attempt allowed within budget |
| prior effect present/probable | any | no failover/new attempt |

Failover is allowed only after the prior provider effect is proven absent.

## Algorithms / Logic

Routing selects an enabled binding by deterministic policy/version and current health before attempt start.

Outcome normalization:
- known accepted -> `PROVIDER_ACCEPTED`
- proven final delivery -> `DELIVERED`
- proven permanent rejection -> `FAILED_PERMANENT`
- proven no-side-effect transient failure -> retryable
- side effect may have occurred -> `UNKNOWN`

`UNKNOWN` rules:
1. no blind retry with a new identity
2. no provider failover while unresolved
3. retry is allowed only when same provider operation is duplicate-safe under the same stable delivery identity
4. otherwise run reconciliation when provider supports it
5. non-reconcilable unknown parks for bounded operator/policy resolution
6. operator resolution records one of `EFFECT_PRESENT`, `EFFECT_ABSENT`, or `RISK_ACCEPTED_NO_RETRY` with evidence/reason; it cannot directly fabricate `DELIVERED`
7. only `EFFECT_ABSENT` may reopen delivery for a new attempt; `EFFECT_PRESENT` advances only to the strongest provider-normalized state supported by evidence
8. historical attempt evidence is never rewritten

Known transient retries use exponential backoff with full jitter, provider `Retry-After` when valid, and bounded max attempts/age. Failover is allowed only after a proven non-side-effect failure and routing policy permits it.

## Configuration

Per Channel Profile declares:
- max attempts
- max delivery age
- base/max backoff
- provider priority/failover eligibility
- concurrency/rate limits
- circuit-breaker thresholds
- capability flags validated against adapter declaration

Default circuit breaker opens after a configured rolling failure threshold and probes with bounded half-open concurrency.

## Security Notes

Secret values are resolved only for the provider call and held in memory for the minimum scope. Provider endpoint identities are configuration, not caller URLs. PII in provider errors is normalized/redacted before persistence/telemetry.

## Failure Handling

Process crash after durable attempt STARTED is treated as potentially unknown unless the provider contract proves the request was never sent. Circuit breaker/open provider state delays ready work or selects a safe alternative only when previous outcomes are proven.

A provider outage must not turn accepted Notifications into lost state.

## Observability

Metrics:

- `notification_provider_attempt_total{provider,channel,outcome}`
- `notification_provider_latency_seconds`
- `notification_provider_unknown_total`
- `notification_provider_reconciliation_total{outcome}`
- `notification_provider_circuit_state`
- `notification_provider_retry_total{reason}`
- `notification_provider_failover_total`

No raw endpoint PII/secrets in labels.

## Performance Notes

Bulkheads are per provider/channel/Tenant. Provider I/O is outside DB transactions. Worker concurrency respects provider rate limits and shared-platform quotas.

## Testing Strategy

Provider contract tests cover stable idempotency, timeout before/after provider acceptance, non-idempotent ambiguous send, unknown parking, reconciliation proving present/absent, callback resolution, safe failover, Retry-After, circuit breaker, secret rotation before next attempt, and immutable attempt evidence.

## Operational Notes

Operator UI clearly distinguishes `UNKNOWN`, `PROVIDER_ACCEPTED`, and `DELIVERED`. Manual resolution requires reason/evidence and cannot delete prior attempts. Provider capability changes are versioned and do not retroactively alter historical safety decisions.

## Traceability

Implements SAD-005 Provider Adapters, Provider Capability/Suppression/Unknown-Outcome Policy, Retry & Reconciliation, and §4.10. Conforms to PAD-PLT-005 Unknown Provider Outcome and attempt-level late-binding policies.
