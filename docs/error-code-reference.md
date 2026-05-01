# Error code reference

## Purpose
External APIs need stable, documented error codes so clients can handle failures predictably.

This reference covers:
- authentication failures,
- token lifecycle failures,
- scope and permission failures,
- company/resource boundary failures,
- validation failures,
- idempotency conflicts,
- rate limiting,
- webhook management failures,
- internal failures without leaking internal implementation details.

Error codes do not replace audit logging, monitoring, or operational investigation. They provide a stable client-facing contract while internal diagnostics remain separate.

## Integration guide
This reference defines stable error semantics. `docs/integration-guide.md` describes how clients should handle these errors operationally across authentication, idempotency, rate limits, webhooks, token lifecycle, and support workflows.

## Design goals
- **Stable client-facing error codes**: clients can build predictable handling logic.
- **Consistent HTTP status mapping**: similar failures use the same status and error code.
- **No internal exception leakage**: responses do not expose stack traces, service names, database errors, queue names, or vendor details.
- **Company-boundary safe responses**: errors avoid confirming resources outside the resolved company boundary.
- **Actionable but minimal messages**: clients get enough guidance without internal detail.
- **`requestId` included for support and investigation**: clients can provide a safe correlation value.
- **Predictable retry semantics**: clients understand when to retry and when to fix credentials or request data.
- **Separation between external errors and internal diagnostics**: client errors remain stable even when internal implementation changes.

## Non-goals
- Exposing internal stack traces.
- Exposing internal service names.
- Modelling every possible product-domain error.
- Replacing audit logs.
- Replacing support processes.
- Defining legal/compliance obligations.
- Providing production-ready code.

## Error response shape
Conceptual response:
```json
{
  "error": "insufficient_scope",
  "message": "The token does not include the required scope for this endpoint.",
  "requestId": "req_123",
  "details": {
    "requiredScope": "payments:create"
  }
}
```

Fields:
- `error`: stable, machine-readable error code.
- `message`: safe and concise human-readable message.
- `requestId`: request reference for support and investigation.
- `details`: optional structured context for validation or conflict cases.

Rules:
- `error` must be stable and machine-readable.
- `message` should be safe and concise.
- `requestId` should be present when available.
- `details` should be structured, minimal, and safe.
- `details` must not contain raw secrets, sensitive payloads, or internal stack traces.

## HTTP status mapping
| HTTP status | Meaning | Typical error codes | Notes |
|---|---|---|---|
| `400 Bad Request` | Request could not be parsed or is unsupported. | `invalid_request` | Use when the request shape is malformed or unsupported. |
| `401 Unauthorized` | Authentication failed or token lifecycle state blocks authentication. | `authentication_failed`, `token_expired`, `token_revoked`, `token_suspended`, `token_not_active` | Do not echo raw API keys or token fingerprints to clients. |
| `403 Forbidden` | Caller is authenticated but not authorized for the operation or boundary. | `insufficient_scope`, `endpoint_not_allowed`, `company_boundary_violation` | Avoid confirming resources outside the resolved company boundary. |
| `404 Not Found` | Resource is not visible under the resolved company boundary. | `resource_not_found` | Prefer this for invisible resources to avoid cross-company existence leaks. |
| `409 Conflict` | Request conflicts with existing idempotency state or resource state. | `idempotency_conflict` | Include only safe conflict context. |
| `422 Unprocessable Entity` | Request is syntactically valid but field validation failed. | `validation_failed`, `unsupported_currency`, `unsupported_country`, `invalid_webhook_url`, `webhook_event_type_not_supported` | Keep field details minimal and external-contract based. |
| `429 Too Many Requests` | Request exceeded a rate limit. | `rate_limit_exceeded` | Include retry guidance only when safe. |
| `500 Internal Server Error` | Unexpected platform failure. | `internal_error` | Do not expose internal implementation details. |
| `503 Service Unavailable` | Temporary platform unavailability. | `service_unavailable` | Retry guidance may be provided when safe. |

## Error code catalogue
| Error code | HTTP status | Category | Client action | Audit expectation | Notes |
|---|---|---|---|---|---|
| `authentication_failed` | `401` | Authentication / token | Provide a valid API key. | May be audited with minimized token reference when available. | Use for missing, malformed, or unknown API key. |
| `token_expired` | `401` | Authentication / token | Rotate or request a new token. | Audit denied lifecycle decision. | Expiry must be enforced at request time. |
| `token_revoked` | `401` | Authentication / token | Stop using the token and contact the token owner. | Audit denied lifecycle decision and consider alerting. | Revocation blocks future authorization. |
| `token_suspended` | `401` | Authentication / token | Contact the token owner or support path. | Audit denied lifecycle decision. | Suspension is temporary disablement. |
| `token_not_active` | `401` | Authentication / token | Activate or replace the token. | Audit denied lifecycle decision when token is resolved. | Use for pending or otherwise non-active tokens. |
| `insufficient_scope` | `403` | Authorization | Request the required scope through the expected operational process. | Audit authorization denial. | Include `requiredScope` only when safe and documented. |
| `endpoint_not_allowed` | `403` | Authorization | Do not call this endpoint with the token. | Audit authorization denial. | Use when endpoint access is not permitted by contract or policy. |
| `company_boundary_violation` | `403` | Authorization | Remove caller-provided company override or correct the request. | Audit boundary denial. | Use safe wording to avoid exposing other company data. |
| `resource_not_found` | `404` | Authorization | Verify resource identifier and company ownership. | May audit boundary-related denial. | Use when resource is not visible under resolved company boundary. |
| `validation_failed` | `422` | Validation | Fix field-level validation errors. | Operational logging may be enough unless sensitive or repeated. | `details` may include safe field names and messages. |
| `invalid_request` | `400` | Validation | Fix malformed or unsupported request shape. | Operational logging may be enough. | Use for invalid JSON, unsupported content type, or unsupported structure. |
| `unsupported_currency` | `422` | Validation | Use a supported currency for the endpoint. | Operational logging may be enough. | Keep supported values aligned with the external contract. |
| `unsupported_country` | `422` | Validation | Use a supported country for the endpoint. | Operational logging may be enough. | Avoid exposing internal availability logic. |
| `invalid_webhook_url` | `422` | Validation | Provide a valid webhook URL. | Audit validation failure for webhook management where meaningful. | Use for URL format or policy validation failure. |
| `idempotency_key_required` | `400` | Idempotency | Include an `Idempotency-Key` header. | Operational logging may be enough. | Use for write operations that require idempotency. |
| `idempotency_conflict` | `409` | Idempotency | Do not reuse the same key with a conflicting payload. | Audit or log conflict where meaningful. | Key scope should include token, company, and endpoint. |
| `rate_limit_exceeded` | `429` | Rate limiting | Retry after backoff. | Audit when meaningful, especially for repeated throttling. | Do not expose internal capacity details. |
| `webhook_target_rejected` | `422` | Webhook | Provide an allowed webhook target. | Audit webhook target rejection. | Use for targets rejected by platform policy. |
| `webhook_event_type_not_supported` | `422` | Webhook | Use a documented event type. | Operational logging may be enough. | Do not expose internal event names. |
| `webhook_disabled` | `403` | Webhook | Re-enable or replace the webhook through the management flow. | Audit webhook management or delivery denial. | Use when webhook status blocks management or delivery action. |
| `internal_error` | `500` | Server / platform | Retry cautiously depending on operation idempotency. | Correlate internally by `requestId`. | Generic response only. |
| `service_unavailable` | `503` | Server / platform | Retry with backoff. | Correlate internally by `requestId`. | Use for temporary platform unavailability. |

## Authentication and token lifecycle errors
Invalid or missing API key should return `authentication_failed`.

Expired token should return `token_expired`. Revoked token should return `token_revoked`. Suspended token should return `token_suspended`. Pending or non-active token should return `token_not_active`.

Raw API key should never be echoed. Denied lifecycle decisions should be audited.

## Authorization and company-boundary errors
Missing required scope should return `insufficient_scope`. Endpoint not allowed should return `endpoint_not_allowed`.

Caller-provided `companyId` mismatch should return `company_boundary_violation` or safe generic forbidden behavior.

Invisible resources should usually return `resource_not_found` to avoid leaking existence across company boundaries.

Denied authorization decisions should be audited.

## Validation errors
Use `validation_failed` for field-level input problems. Use `invalid_request` for malformed or unsupported request shape.

Use `unsupported_currency` and `unsupported_country` for stable external validation cases. Use `invalid_webhook_url` for webhook target validation.

Validation details must not expose internal rules unnecessarily.

## Idempotency errors
Write operations may require an `Idempotency-Key`.

Missing key should return `idempotency_key_required`. Reused key with conflicting payload should return `idempotency_conflict`.

Idempotency must be scoped to token, company, and endpoint. Idempotency errors should include `requestId`.

## Rate-limit errors
`rate_limit_exceeded` maps to `429`.

Include `retryAfterSeconds` when safe and available. Do not expose internal capacity or exact policy internals.

Rate-limit outcomes should be auditable when meaningful.

## Webhook errors
Invalid target should return `invalid_webhook_url` or `webhook_target_rejected`.

Unsupported event type should return `webhook_event_type_not_supported`.

Disabled webhook management should use `webhook_disabled` where appropriate.

Webhook-related errors should avoid exposing internal delivery infrastructure.

## Server and platform errors
`internal_error` should be generic. `service_unavailable` should be used for temporary platform unavailability.

Do not expose stack traces, internal service names, database errors, queue names, or vendor details.

Include `requestId` for support correlation.

## Retry semantics
| Error code | Retry? | Client guidance |
|---|---|---|
| `authentication_failed` | no | Retry only after providing a valid credential. |
| `token_expired` | no | Rotate or request a new token before retrying. |
| `token_revoked` | no | Do not retry with the revoked token. |
| `token_suspended` | no | Retry only after explicit reactivation. |
| `token_not_active` | no | Retry only after activation or replacement. |
| `insufficient_scope` | no | Retry only after scope assignment changes. |
| `endpoint_not_allowed` | no | Do not retry unless endpoint access changes. |
| `company_boundary_violation` | no | Retry only after correcting company/resource input. |
| `resource_not_found` | conditional | Retry only if resource ownership or identifier changes. |
| `validation_failed` | no | Fix request fields before retrying. |
| `invalid_request` | no | Fix request shape before retrying. |
| `unsupported_currency` | no | Use a supported currency before retrying. |
| `unsupported_country` | no | Use a supported country before retrying. |
| `invalid_webhook_url` | no | Provide an allowed webhook URL before retrying. |
| `idempotency_key_required` | no | Add an idempotency key before retrying. |
| `idempotency_conflict` | no | Do not retry with the same conflicting payload. |
| `rate_limit_exceeded` | yes | Retry after backoff or `retryAfterSeconds` when provided. |
| `webhook_target_rejected` | no | Provide an allowed target before retrying. |
| `webhook_event_type_not_supported` | no | Use a documented event type before retrying. |
| `webhook_disabled` | no | Retry only after webhook status changes. |
| `service_unavailable` | yes | Retry with backoff. |
| `internal_error` | conditional | Retry cautiously depending on operation idempotency. |

## Audit expectations
Authentication failures may be audited with minimized token reference when available.

Authorization denials, company-boundary violations, and token lifecycle denials should be audited.

Rate-limit exceeded events should be audited when meaningful.

Validation errors may be logged operationally but should avoid sensitive payloads.

Internal errors should be correlated with `requestId`, not leaked to clients.

## Common mistakes
- Returning raw internal exception messages.
- Using inconsistent error codes for the same condition.
- Revealing that another company's resource exists.
- Omitting `requestId`.
- Exposing stack traces.
- Exposing raw API key or token fingerprint to clients.
- Returning `200` with error payload.
- Using only generic `internal_error` for all failures.
- Making retry behavior ambiguous.
- Placing sensitive request payloads in `details`.

## Implementation notes
This model is language-agnostic and conceptual. It does not define application code, framework boilerplate, or an OpenAPI/YAML file.

Teams may later translate this reference into API portal docs, OpenAPI error components, SDK error types, or integration guides.
