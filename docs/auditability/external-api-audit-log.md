# External API Audit Log

This document describes an illustrative audit model for company-scoped external API access in a fintech-style reference architecture. It is a non-production reference and stays at the design level: it does not prescribe a vendor-specific logging tool, queue, database, cloud service, SIEM product, or retention policy.

Audit logs should help explain which external API decision was made, for which token owner, under which effective `CompanyId`, against which endpoint, and why the platform allowed, denied, or rate-limited the request.

## Related Documents

- [Request flow diagrams](../architecture/request-flow.md)
- [API key lifecycle](../token-lifecycle/api-key-lifecycle.md)
- [Permission model](../scopes/permission-model.md)
- [Threat model](../threat-model.md)
- [External API rate limiting](../rate-limiting/external-api-rate-limiting.md)
- [Internal vs external API](../integration-boundaries/internal-vs-external-api.md)
- [Error code reference](../error-code-reference.md)
- [Minimal illustrative OpenAPI contract](../../examples/openapi/external-api.sample.yaml)

## Why Auditability Matters

External API requests combine authentication, company ownership, scope checks, rate-limit decisions, product data access, and stable client-facing errors. Auditability matters because operators need to answer questions such as:

- Which token identifier was used?
- Which company owned that token?
- Which effective `CompanyId` was applied?
- Which endpoint and required scope were involved?
- Was the request allowed, denied, rate-limited, or blocked by token lifecycle state?
- Which correlation ID connects the client response to internal diagnostics?
- Did a denied request indicate token misuse, over-broad scopes, company-boundary probing, or retry abuse?

Audit events should support investigation and review without turning the audit log into a store for raw secrets or sensitive payloads.

## Request-Time Placement

Audit logging appears in the request flow after the platform has enough context to describe the decision safely.

For successful requests, the audit event should include token identifier, token owner, effective company scope, required scope, endpoint, decision result, response status, and correlation ID.

For credential failures, the platform may not have an effective `CompanyId`. The audit event should still record a safe credential-denial classification, token prefix or token identifier when available, endpoint, response status, and correlation ID.

See the [request flow diagrams](../architecture/request-flow.md) for successful, credential-denied, scope-denied, and rate-limited request paths.

## Events to Audit

The audit model should cover request-time decisions and token lifecycle changes.

Request-time events:

- successful external API request,
- missing API key,
- unknown API key or unknown key prefix,
- invalid token verification,
- expired token attempt,
- revoked token attempt,
- deactivated or non-active token attempt,
- missing required scope,
- company-scope violation or resource outside resolved company scope,
- endpoint not allowed by the external contract,
- rate-limit exceeded,
- abuse-control denial,
- sensitive write operation such as a payment initiation request.

Token lifecycle events:

- token created,
- raw token displayed once,
- token activated,
- token scopes changed,
- token rotated,
- token expired,
- token revoked,
- token deactivated,
- token reactivated.

External contract and boundary events can also be audited when useful, such as repeated calls to unknown or internal-looking routes. Those events should use safe route labels and avoid leaking internal implementation details.

## Illustrative Audit Event Schema

The following generic schema shows the kind of metadata an audit event can contain. Field names are illustrative, not a required implementation format.

```json
{
  "event_id": "audit_evt_123",
  "event_type": "external_api.request.denied_scope",
  "occurred_at": "2026-01-01T12:00:00Z",
  "correlation_id": "req_123",
  "decision": "denied",
  "error_code": "insufficient_scope",
  "http_status": 403,
  "token": {
    "id": "token_123",
    "key_prefix": "key_demo",
    "owner_company_id": "company_123",
    "status": "active"
  },
  "company": {
    "effective_company_id": "company_123",
    "caller_provided_company_id": "company_other"
  },
  "request": {
    "method": "POST",
    "route_template": "/external/v1/payments",
    "required_scope": "payments:create",
    "client_ip": "203.0.113.10",
    "user_agent": "ExampleClient/1.0"
  },
  "rate_limit": {
    "policy_id": "external_payments_write",
    "limited": false
  },
  "metadata": {
    "resource_type": "payment_request",
    "resource_id": "pay_demo_001"
  }
}
```

Use stable identifiers and route templates where possible. Resource identifiers should be included only when they are necessary for investigation and safe for the audience that can view audit records.

## Example Event Types

Use stable, queryable event type names. Examples:

| Event type | Meaning |
|---|---|
| `external_api.request.allowed` | Request passed authentication, company resolution, scope checks, and rate-limit controls. |
| `external_api.request.denied_authentication` | Request was denied because the API key was missing, unknown, invalid, expired, revoked, or non-active. |
| `external_api.request.denied_scope` | Token was valid but lacked the required endpoint scope. |
| `external_api.request.denied_company_boundary` | Request attempted to override company scope or access a resource outside the resolved company. |
| `external_api.request.rate_limited` | Request was rejected by a rate-limit policy. |
| `external_api.request.abuse_denied` | Request was denied by an abuse-control rule. |
| `external_api.token.created` | Token record was created. |
| `external_api.token.displayed_once` | Raw token was displayed through the creation or replacement flow. |
| `external_api.token.activated` | Token became active. |
| `external_api.token.scopes_changed` | Token scope grants were added or removed. |
| `external_api.token.rotated` | Replacement token was issued and linked to rotation. |
| `external_api.token.expired` | Token reached its expiry state. |
| `external_api.token.revoked` | Token was revoked before natural expiry. |
| `external_api.token.deactivated` | Token was temporarily disabled. |
| `external_api.token.reactivated` | Token was re-enabled after an explicit action. |

The exact names can vary, but they should remain stable enough for review, dashboards, and investigation.

## Metadata to Capture

External API audit events should capture safe metadata such as:

- timestamp,
- event type,
- decision result,
- error code or error classification where applicable,
- HTTP status,
- request correlation ID,
- token identifier or key prefix,
- token owner company,
- effective `CompanyId`,
- caller-provided company identifier when relevant and safe,
- HTTP method,
- route template,
- required scope,
- granted or missing scope context when safe,
- rate-limit policy identifier when applicable,
- client IP address from the trusted edge layer,
- user agent or integration identifier when useful,
- lifecycle actor or initiating system for token management events,
- safe resource type and resource identifier when needed for investigation.

Prefer route templates over raw paths, such as `/external/v1/payments/{payment_id}` instead of a full path with a specific identifier, unless the identifier is explicitly needed and safe to include.

## Token Owner and Effective CompanyId

Audit events should distinguish token ownership from caller input:

- `owner_company_id` identifies the company stored on the token record.
- `effective_company_id` identifies the company scope applied to the request.
- `caller_provided_company_id` can record untrusted request input when useful for investigation.

For valid active tokens, `owner_company_id` and `effective_company_id` should normally match because the token owner determines effective company scope. Caller-provided company identifiers must not replace the resolved company scope.

For missing, unknown, malformed, or hash-mismatched keys, token owner and effective company fields may be unavailable. In those cases, audit the denial with a safe token prefix or request fingerprint only when available.

## Scope Checks and Denial Outcomes

Scope checks happen after token validation and company resolution. Audit events for scope decisions should include:

- required scope,
- route template,
- token identifier or key prefix,
- owner company,
- effective `CompanyId`,
- decision result,
- error code such as `insufficient_scope` when denied,
- response status such as `403`.

Do not include the full list of token scopes in every request event unless there is a clear review need. Scope grants and removals should be auditable as lifecycle or administration events.

## Token Credential Denials

Expired, revoked, unknown, invalid, deactivated, and missing-token attempts should be auditable because they can indicate stale integrations, leaked tokens, replay attempts, or probing.

Represent these events with stable classifications such as:

- `missing_api_key`,
- `unknown_api_key`,
- `invalid_api_key`,
- `expired_api_key`,
- `revoked_api_key`,
- `deactivated_api_key`,
- `token_not_active`.

Client-facing errors should remain stable and safe. The [error code reference](../error-code-reference.md) describes external error semantics such as `authentication_failed`, `token_expired`, `token_revoked`, and `token_not_active`.

## Rate-Limit and Abuse-Control Denials

Rate-limit and abuse-control events should make it possible to understand which policy was involved without exposing internal capacity details.

Capture safe metadata such as:

- token identifier or key prefix,
- owner company,
- effective `CompanyId`,
- route template,
- operation type,
- policy identifier,
- decision result such as `rate_limited` or `denied`,
- response status such as `429`,
- retry guidance only when safe and already exposed to the client,
- correlation ID.

Rate-limit audit events should not reveal exact internal thresholds, capacity, queue depth, or other clients' usage.

## Lifecycle Events

Lifecycle audit events should describe changes to token state and authorization metadata.

Useful lifecycle metadata includes:

- token identifier or key prefix,
- owner company,
- lifecycle action,
- previous status and new status when relevant,
- added or removed scopes for scope changes,
- expiry timestamp when relevant,
- replacement token identifier for rotation when safe,
- actor or initiating system where available,
- timestamp,
- correlation ID or change reference where useful.

Lifecycle events should never include raw token values. A `displayed_once` event can record that one-time display occurred without storing the token itself.

## What Not to Log

Audit logs must not contain:

- raw API key tokens,
- token hashes,
- full sensitive request bodies,
- full sensitive response bodies,
- secrets, credentials, or signing keys,
- unnecessary personal or financial data,
- internal stack traces,
- internal service names or vendor-specific implementation details,
- sensitive operational state such as exact internal capacity.

When deeper troubleshooting is needed, use separate diagnostic channels with appropriate access controls and redaction rather than expanding audit event payloads.

## Decision Results and Error Codes

Use stable decision results that are easy to query.

| Decision | Meaning |
|---|---|
| `allowed` | Request passed authentication, token lifecycle, company resolution, scope checks, and rate-limit controls. |
| `denied` | Request failed authentication, authorization, company-boundary, validation, or policy checks. |
| `rate_limited` | Request was rejected by a rate-limit policy. |
| `lifecycle_changed` | Token metadata or lifecycle status changed. |

Pair decision results with stable error codes or classifications where applicable. Client-facing error codes and audit error classifications do not need to be identical, but they should be mappable during investigation.

## Review Use Cases

The audit model should support questions such as:

- Which tokens accessed a specific endpoint during a time window?
- Which denied requests attempted to use expired, revoked, unknown, or invalid keys?
- Which company scopes are seeing repeated missing-scope denials?
- Which tokens are repeatedly rate-limited?
- Was a payment initiation request made through an external API key?
- Did a token continue to be used after revocation or expiry?
- Are company-boundary violations clustered by token, endpoint, or source?
- Are external clients receiving stable error responses with correlation IDs?

## Design-Level Review Checklist

- [ ] Are allowed, denied, rate-limited, and lifecycle events auditable?
- [ ] Do request events include token identifier, token owner, effective `CompanyId`, endpoint, required scope, decision, and correlation ID when available?
- [ ] Are missing, unknown, invalid, expired, revoked, and deactivated key attempts classified safely?
- [ ] Are missing-scope and company-boundary denials represented separately?
- [ ] Are rate-limit and abuse-control denials represented without exposing internal capacity details?
- [ ] Are token lifecycle changes and scope changes auditable?
- [ ] Are raw tokens, token hashes, sensitive payloads, and internal implementation details excluded?
