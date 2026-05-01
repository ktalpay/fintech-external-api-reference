# External API Audit Log

External API access should produce audit records for security-relevant decisions and lifecycle events. The audit log should help answer who called what, for which company scope, with which token identifier, and what decision the platform made.

Audit records should be structured, append-only, and safe to inspect during support, security review, and incident response.

## What to Log

External API audit events should include:

- timestamp,
- request correlation ID,
- token identifier or key prefix,
- resolved `CompanyId`,
- external client identifier when available,
- client IP address,
- HTTP method,
- endpoint path or route template,
- required scope,
- decision result,
- error classification when applicable,
- response status code,
- rate-limit policy identifier when applicable,
- user agent or integration identifier when useful,
- lifecycle actor for token management events.

For resource-specific actions, log stable resource identifiers only when they are necessary for investigation and are safe for the audience that can view audit records.

## What Not to Log

Audit logs must not contain:

- raw API key tokens,
- token hashes,
- full sensitive request bodies,
- full sensitive response bodies,
- secrets, credentials, or signing keys,
- unnecessary personal or financial data,
- internal stack traces exposed as client-facing errors.

When detailed troubleshooting is needed, use short-lived, access-controlled diagnostic logs with appropriate redaction rather than expanding the audit log payload.

## Token Identifier vs Raw Token

Audit records should reference a non-secret token identifier, such as:

- API key record `id`,
- key prefix,
- external-facing key label,
- token lifecycle event ID.

The raw token should never appear in audit logs. The token hash should also be excluded because it is an authentication verification artifact, not an audit display field.

## CompanyId

Audit events should store the `CompanyId` resolved from API key metadata. This value is the trusted company scope for the request.

If a request includes a client-provided company identifier, audit it only as request input when needed for investigation. It must not replace the resolved company scope.

## Client IP

Client IP can support abuse detection, allowlist review, and incident investigation. It should be recorded consistently from the trusted gateway layer.

If the deployment uses proxies or load balancers, the platform should define which forwarded headers are trusted and where IP normalization occurs.

## Endpoint

Prefer route templates over raw paths when possible. For example, log `/external/v1/payments/{payment_id}` rather than a path containing a specific payment identifier, unless the identifier is explicitly needed and safe to expose in audit tooling.

## Request Correlation ID

Every external request should have a correlation ID. The same ID should appear in:

- API gateway logs,
- application logs,
- audit events,
- client-facing response headers where appropriate,
- support tooling.

Correlation IDs make it possible to connect an external support report to internal diagnostic records without exposing raw secrets.

## Decision Result

Use stable decision result values. Recommended values:

| Result | Meaning |
|---|---|
| `allowed` | Request passed authentication, authorization, lifecycle, and rate-limit checks. |
| `denied` | Request failed authentication, authorization, company-scope, or validation policy. |
| `rate_limited` | Request was rejected by a rate-limit policy. |
| `revoked` | Request used a revoked API key. |
| `expired` | Request used an expired API key. |

These values should be easy to query and should not depend on localized or human-readable error messages.

## Error Classification

Error classification should separate security and operational causes without leaking sensitive detail.

Examples:

- `missing_api_key`,
- `invalid_api_key`,
- `revoked_api_key`,
- `expired_api_key`,
- `deactivated_api_key`,
- `missing_scope`,
- `company_scope_violation`,
- `rate_limit_exceeded`,
- `invalid_request`,
- `internal_error`.

Client-facing messages can remain generic while audit records keep enough classification to support review.

## Security Review Use Cases

Audit logs should support questions such as:

- Which keys accessed a specific endpoint during a time window?
- Which denied requests attempted to use revoked or expired keys?
- Which company scopes are hitting rate limits most often?
- Which external clients are seeing repeated missing-scope errors?
- Was a sensitive operation performed through an external API key?
- Did a token continue to be used after revocation?
- Are denied requests clustered by IP address, endpoint, or token prefix?

These use cases should influence field design and retention policy, but the audit log should still minimize sensitive data.
