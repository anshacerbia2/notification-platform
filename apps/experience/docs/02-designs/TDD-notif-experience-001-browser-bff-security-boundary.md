---
doc_meta:
  id: TDD-notif-experience-001
  title: Notification Experience Browser and BFF Security Boundary
  owner: Notification Platform Team
  version: 1.0.0
  status: approved
  classification: restricted
  parent_sad: SAD-015
  review_cycle_days: 180
  created_date: 2026-08-27
  last_reviewed: 2026-08-27
---
# Notification Experience Browser and BFF Security Boundary

## Purpose

Define Notification Experience same-origin Go BFF security, OIDC session handling, context binding, CSRF/Origin/Host controls, provider-secret write-only mediation, step-up, template-preview isolation, and telemetry privacy.

## Scope

Covers browser/BFF trust boundary. React feature workflows are TDD-notif-experience-002; provider secret custody and runtime delivery remain outside Experience authority.

## Technical Context

Browser JavaScript communicates only with Notification Experience origin. It never contacts providers, Scheduling, brokers, Notification PostgreSQL, or secret stores. Access/refresh tokens and provider credentials are not exposed to browser JavaScript.

## Component Design

```text
Browser
 -> Go BFF
    -> Session Manager
    -> OIDC Adapter
    -> Context Guard
    -> CSRF/Origin Guard
    -> Notification API Adapter
    -> Secret-write Redaction Guard
```

Opaque cookie `__Host-scnehaux_notif_session` is Secure, HttpOnly, Path=/, no Domain, SameSite=Lax by default.

## Data Model

Server session contains opaque ID, Principal, assurance, current application/Tenant context, context generation, delegated token material/handle, expiry, and CSRF secret hash.

Secret-entry request bodies are never persisted in session state, retry queues, analytics, or request-body logs.

## API / Interface

Auth/context routes mirror Scheduling Experience. Notification-mediated routes are under `/api/notification/*`.

Credential submission endpoint is a dedicated streaming/redaction path. It forwards once to the governed Runtime credential-registration API and returns only secret-reference metadata.

Template preview is rendered in a sandboxed iframe with no scripts, no same-origin privilege, blocked navigation/forms, and restrictive CSP. Active script content is stripped/rejected.

## Algorithms / Logic

OIDC uses Authorization Code + PKCE, state, and nonce. Session rotates after login, step-up, privilege change, and context switch.

Context switch increments generation and forces browser cancellation/cache/form reset.

Secret-write flow:
1. step-up if policy requires
2. receive bounded credential field over TLS
3. mark request body non-loggable/non-retriable
4. forward server-to-server
5. zero/drop in-memory buffer as soon as practical
6. return masked reference metadata only

## Configuration

- session absolute 8 h
- idle 30 min
- step-up validity 10 min
- secret request max 64 KiB
- template preview max rendered 1 MiB
- strict allowed origins/hosts
- no wildcard authenticated CORS

## Security Notes

CSP, frame-ancestors, HSTS, safe redirects, cookie flags, CSRF, Origin/Host checks, and output encoding are mandatory. Preview iframe receives no privileged scripts or credential-bearing fetch access.

Provider secrets, access tokens, refresh tokens, recipient payloads, rendered communication bodies, and CSRF values are redacted from telemetry.

## Failure Handling

Secret submission is not automatically replayed after ambiguous upstream failure; UI asks operator to inspect resulting reference state/retry explicitly. Session/context failures clear unsafe state. Preview rendering failure cannot execute fallback HTML in the admin origin.

## Observability

Metrics cover auth/session, CSRF/origin rejection, context switch, step-up, secret-registration outcome without values, preview-sandbox violations, and upstream API latency.

## Performance Notes

BFF streams bounded secret/body requests and does not retain content. Static assets are immutable/cacheable. Large delivery histories remain paginated server-side.

## Testing Strategy

Security/E2E tests cover session fixation, CSRF, Origin/Host, context stale tabs, step-up, secret non-log/non-cache/non-return, browser storage scan, template XSS/script/form/navigation payloads, CSP/sandbox, redirect abuse, token leakage, and no direct provider/Scheduling/broker/DB access.

## Operational Notes

Experience outage does not stop Notification delivery. Any suspected secret exposure triggers session/log/cache inspection runbook and credential rotation through Runtime/secret authority.

## Traceability

Implements SAD-015 authentication/session, browser security, write-only provider-secret, sandboxed preview, and same-origin BFF decisions.
