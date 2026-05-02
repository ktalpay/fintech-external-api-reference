# Error Code Reference

This document describes an illustrative error model for company-scoped external API access in a fintech-style reference architecture. It is a non-production reference for stable external error behavior, not a runtime implementation guide.

Stable external error codes help clients decide whether to fix credentials, request more scope, correct request data, retry later, or contact an operational support path. They also help operators connect client-facing responses to audit events without exposing internal implementation details.

## Related Documents

- [Minimal illustrative OpenAPI contract](../examples/openapi/external-api.sample.yaml)
- [Request flow diagrams](architecture/request-flow.md)
- [API key lifecycle](token-lifecycle/api-key-lifecycle.md)
- [Permission model](scopes/permission-model.md)
- [External API audit log](auditability/external-api-audit-log.md)
- [External API rate limiting](rate-limiting/external-api-rate-limiting.md)
- [Threat model](threat-model.md)

## Design Goals

- Use stable machine-readable error codes.
- Keep client-facing messages safe and concise.
- Include a request correlation value such as `requestId`.
- Avoid leaking raw API keys, token hashes, stack traces, internal service names, vendor details, or sensitive payloads.
- Avoid confirming resources outside the resolved company scope.
- Keep external responses minimal while audit logs can carry richer internal context for investigation.

## Stable External Error Shape

The illustrative OpenAPI sample uses this shape:

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

- `error`: stable machine-readable error code.
- `message`: safe human-readable message.
- `requestId`: request correlation identifier for support and audit lookup.
- `details`: optional safe structured context for validation or authorization errors.

`details` must remain minimal. It must not include raw secrets, full sensitive request bodies, full sensitive response bodies, internal stack traces, or internal implementation details.

## HTTP Status Mapping

| HTTP status | Meaning | Typical error codes |
|---|---|---|
| `400 Bad Request` | Request could not be parsed or is unsupported. | `invalid_request` |
| `401 Unauthorized` | Authentication failed or token lifecycle state blocks access. | `authentication_failed`, `token_expired`, `token_revoked`, `token_not_active` |
| `403 Forbidden` | Caller is authenticated but not authorized for the operation or company boundary. | `insufficient_scope`, `endpoint_not_allowed`, `company_boundary_violation` |
| `404 Not Found` | Resource is not visible inside the resolved company scope. | `resource_not_found` |
| `409 Conflict` | Request conflicts with idempotency state or resource state. | `idempotency_conflict` |
| `422 Unprocessable Entity` | Request is syntactically valid but field validation failed. | `validation_failed`, `unsupported_currency`, `unsupported_country`, `invalid_webhook_url` |
| `429 Too Many Requests` | Request exceeded a rate limit or throttling policy. | `rate_limit_exceeded` |
| `500 Internal Server Error` | Unexpected platform-side failure. | `internal_error` |
| `503 Service Unavailable` | Temporary platform-side unavailability. | `service_unavailable` |

## Core Error Codes

| Error code | HTTP status | Category | Client guidance | Audit expectation |
|---|---|---|---|---|
| `authentication_failed` | `401` | Authentication | Provide a valid API key. | May be audited with minimized token reference when available. |
| `token_expired` | `401` | Token lifecycle | Rotate or request a replacement token before retrying. | Audit denied lifecycle decision. |
| `token_revoked` | `401` | Token lifecycle | Stop using the revoked token. | Audit denied lifecycle decision. |
| `token_not_active` | `401` | Token lifecycle | Retry only after activation, reactivation, or replacement. | Audit denied lifecycle decision when token is resolved. |
| `insufficient_scope` | `403` | Authorization | Request the required scope through the expected operational process. | Audit authorization denial. |
| `endpoint_not_allowed` | `403` | Authorization | Do not call this endpoint unless external contract access changes. | Audit authorization denial. |
| `company_boundary_violation` | `403` | Authorization | Remove caller-provided company override or correct the resource reference. | Audit company-boundary denial. |
| `resource_not_found` | `404` | Authorization / resource visibility | Verify the resource identifier and company-scoped visibility. | May audit boundary-related denial. |
| `invalid_request` | `400` | Validation | Fix malformed JSON, unsupported content type, or unsupported request shape. | Operational logging may be enough unless repeated or suspicious. |
| `validation_failed` | `422` | Validation | Fix request fields before retrying. | Operational logging may be enough unless sensitive or repeated. |
| `unsupported_currency` | `422` | Validation | Use a supported currency for the endpoint. | Operational logging may be enough. |
| `unsupported_country` | `422` | Validation | Use a supported country for the endpoint. | Operational logging may be enough. |
| `invalid_webhook_url` | `422` | Validation | Provide a valid webhook URL. | Audit when meaningful for webhook management review. |
| `idempotency_key_required` | `400` | Idempotency | Include an idempotency key for the write operation. | Operational logging may be enough. |
| `idempotency_conflict` | `409` | Idempotency | Do not reuse the same idempotency key with a conflicting payload. | Audit or log where meaningful. |
| `rate_limit_exceeded` | `429` | Rate limiting | Retry after backoff or documented retry guidance. | Audit when meaningful, especially for repeated throttling. |
| `internal_error` | `500` | Platform | Retry cautiously depending on operation idempotency. | Correlate internally by `requestId`; do not expose details. |
| `service_unavailable` | `503` | Platform | Retry with backoff. | Correlate internally by `requestId`; do not expose details. |

The [OpenAPI sample](../examples/openapi/external-api.sample.yaml) intentionally shows a small subset of these codes: `authentication_failed`, `token_expired`, `token_revoked`, `insufficient_scope`, `company_boundary_violation`, `resource_not_found`, `validation_failed`, and `rate_limit_exceeded`.

## Authentication and Token Lifecycle Errors

Missing, malformed, unknown, or invalid API keys should use `authentication_failed`.

Expired tokens should use `token_expired`. Revoked tokens should use `token_revoked`. Deactivated, pending, or otherwise non-active tokens should use `token_not_active`.

Client responses should not echo raw API keys, key prefixes, token hashes, or token fingerprints. Audit logs may record a safe token identifier or key prefix when available.

## Scope and Permission Errors

When a valid token does not include the required endpoint scope, use `insufficient_scope`.

When the endpoint is not allowed by the external contract or policy, use `endpoint_not_allowed`.

Scope errors should be returned only after token validation and company scope resolution. Audit logs should capture the token identifier, effective `CompanyId`, route template, required scope, decision result, and `requestId` when available.

## Company Boundary and Not Found Errors

External callers must not choose arbitrary company scope. The effective company scope comes from the API key token owner.

Use `company_boundary_violation` when the request attempts to override company scope or access outside the resolved company boundary and a forbidden response is appropriate.

Use `resource_not_found` when a resource is not visible inside the resolved company scope. This avoids confirming whether a resource exists for another company.

## Validation Errors

Use `invalid_request` for malformed or unsupported request shapes.

Use `validation_failed` for field-level validation problems in otherwise parseable requests. Optional `details` can include safe field names or concise validation context, such as:

```json
{
  "error": "validation_failed",
  "message": "Request validation failed.",
  "requestId": "req_123",
  "details": {
    "field": "amount.value"
  }
}
```

Validation details should not expose internal rules, sensitive payload values, or implementation internals.

## Rate Limit and Throttling Errors

Use `rate_limit_exceeded` with `429 Too Many Requests` for rate-limit or throttling denials.

Responses can include retry guidance when safe and documented, such as a `Retry-After` header. They should not expose exact internal thresholds, capacity, queue depth, policy internals, or other companies' usage.

Rate-limit outcomes should be auditable when meaningful, especially for repeated throttling, retry loops, invalid-token floods, or suspicious usage patterns.

## Generic Server-Side Errors

Use `internal_error` for unexpected platform-side failures.

Use `service_unavailable` for temporary platform-side unavailability.

These responses should be generic and include `requestId` for correlation. They must not expose stack traces, internal service names, database errors, queue names, vendor details, or sensitive operational state.

## Retry Semantics

| Error code | Retry? | Client guidance |
|---|---|---|
| `authentication_failed` | No | Retry only after providing a valid API key. |
| `token_expired` | No | Rotate or request a replacement token before retrying. |
| `token_revoked` | No | Do not retry with the revoked token. |
| `token_not_active` | No | Retry only after activation, reactivation, or replacement. |
| `insufficient_scope` | No | Retry only after scope assignment changes. |
| `endpoint_not_allowed` | No | Retry only if endpoint access changes. |
| `company_boundary_violation` | No | Correct company or resource input before retrying. |
| `resource_not_found` | Conditional | Retry only if the resource identifier or company-scoped visibility changes. |
| `invalid_request` | No | Fix request shape before retrying. |
| `validation_failed` | No | Fix request fields before retrying. |
| `idempotency_conflict` | No | Do not retry with the same conflicting payload. |
| `rate_limit_exceeded` | Yes | Retry after backoff or documented retry guidance. |
| `internal_error` | Conditional | Retry cautiously depending on operation idempotency. |
| `service_unavailable` | Yes | Retry with backoff. |

## Audit and Diagnostics Boundary

External responses should remain stable and minimal. Audit logs may carry richer internal context, such as token identifier, key prefix, effective `CompanyId`, required scope, route template, decision result, error classification, rate-limit policy category, and correlation ID.

Audit records should still avoid raw API keys, token hashes, full sensitive request bodies, full sensitive response bodies, stack traces, and unnecessary sensitive data.

## Common Mistakes

- Returning raw internal exception messages.
- Exposing raw API keys, token hashes, or token fingerprints.
- Exposing stack traces, internal service names, database errors, queue names, or vendor details.
- Revealing whether another company's resource exists.
- Omitting `requestId`.
- Using inconsistent error codes for the same condition.
- Returning `200` with an error payload.
- Making retry behavior ambiguous.
- Putting sensitive request payloads in `details`.
- Using client-facing responses as the only investigation record instead of audit logs.

## Implementation Notes

This model is conceptual and implementation-oriented. Teams may translate it into API portal docs, OpenAPI error components, SDK error types, or integration guides while keeping the external error shape stable.
