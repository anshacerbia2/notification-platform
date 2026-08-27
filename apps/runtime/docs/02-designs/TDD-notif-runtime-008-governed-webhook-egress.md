---
doc_meta:
  id: TDD-notif-runtime-008
  title: Governed Webhook Egress
  owner: Notification Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-005
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Governed Webhook Egress

## Purpose

Define webhook as a governed outbound communication channel with registered destinations, SSRF-resistant network controls, bounded redirects/payloads/timeouts, TLS verification, authentication secret references, and provider-style delivery evidence.

## Scope

Covers production webhook target registration and outbound HTTP behavior. It is not a generic arbitrary HTTP executor and does not accept per-send free-form URLs.

## Technical Context

Webhook shares Notification Delivery/Attempt lifecycle but has unique SSRF and network-boundary risks. Registered destination metadata is Notification-owned; credentials remain secret references.

## Component Design

```text
Webhook Delivery Adapter
  -> RegisteredWebhookTarget
  -> URL/Policy Validator
  -> Safe Resolver/Dialer
  -> HTTP Client
  -> normalized provider-style outcome
```

The HTTP client disables unsafe implicit redirect behavior and applies policy at every hop.

## Data Model

Registered target contains:
- target ID/version
- owner application/Tenant scope
- HTTPS URL
- allowed redirect policy
- authentication mode + secret reference
- optional governed internal-target class
- payload schema/version
- enabled state
- validation evidence

Attempt evidence freezes target version, normalized endpoint identity, resolved address class, auth reference version, request hash, HTTP status class, and safe response hash/metadata.

## API / Interface

Per-send command references `webhook_target_id`; it cannot supply URL or credentials.

Production external targets require HTTPS. Redirects are disabled by default; when enabled, maximum 3 hops and every new destination is fully revalidated.

Default limits:
- connect timeout 3 s
- total request timeout 10 s
- request body 256 KiB
- response body captured max 64 KiB
- no credential in query string

## Algorithms / Logic

Before connect:

1. parse and normalize URL
2. require allowed scheme/port policy
3. resolve DNS using controlled resolver
4. reject loopback, link-local, multicast, unspecified, and private ranges for external-target class
5. reject cloud metadata/link-local destinations
6. bind connection to the validated resolved address while preserving TLS hostname verification
7. on redirect, repeat all validation; never reuse prior trust decision
8. reject scheme downgrade
9. enforce TLS hostname/certificate verification
10. stream bounded body/response

DNS rebinding defense validates the actual address used for the connection, not only a preflight lookup.

A governed internal-target class may allow explicitly registered CIDRs/hostnames through separate policy; it is not enabled by caller input.

## Configuration

Network deny sets include IPv4/IPv6 loopback, private, link-local, multicast, unspecified, and metadata ranges. Enterprise egress proxy/firewall policy is defense in depth, not a replacement for application validation.

Allowed ports default to 443 for external targets.

## Security Notes

Authorization to create/change targets is privileged. Secret references are write-only through configuration APIs. Authorization headers are stripped on redirects unless the redirect remains within the explicitly registered credential scope.

TLS verification cannot be disabled in production.

## Failure Handling

DNS failure, policy rejection, connect timeout, TLS failure, oversized response, and safe HTTP failure map to normalized Delivery outcomes. Ambiguous timeout after request transmission follows the same `UNKNOWN` rules as other non-idempotent providers unless the target contract supports stable idempotency.

## Observability

Metrics cover target policy rejections, DNS/TLS/connect failures, redirects, latency, status class, and unknown outcomes. Logs never emit Authorization headers, full URLs with sensitive query data, or response bodies.

## Performance Notes

Connection pools are partitioned by safe target identity and bounded. Response bodies are not buffered beyond 64 KiB. Slow targets cannot consume unbounded worker concurrency.

## Testing Strategy

Security tests include localhost/private/link-local/metadata IPv4+IPv6, decimal/octal/IPv6 host forms, DNS rebinding simulation, redirect to denied range, TLS hostname mismatch, downgrade redirect, oversized body/response, credential stripping, timeout ambiguity, target ownership, and registered-internal-class allowlist.

## Operational Notes

Webhook incidents expose normalized target ID/version and network failure class without revealing credentials. Emergency target disablement stops new attempts; already-started external effects follow normal attempt semantics.

## Traceability

Implements SAD-005 Webhook Egress Security and PAD-PLT-005 governed-webhook policy. Delivery/UNKNOWN semantics reuse TDD-004; configuration ownership reuses TDD-003.
