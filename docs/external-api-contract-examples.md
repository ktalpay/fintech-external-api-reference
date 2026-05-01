# External API contract examples

## Purpose
External API contracts should be explicit, stable, and separated from internal APIs. They define what an external client can rely on without exposing internal service structure, internal operational endpoints, or implementation details.

Each contract should define:
- method and path,
- required scope,
- authentication requirement,
- company/resource boundary,
- request shape,
- response shape,
- error model,
- audit expectations,
- rate-limit expectations,
- idempotency behavior where relevant.

## Why contract examples matter
External clients need stable contracts so they can build integrations without depending on internal implementation details.

Internal APIs should not be exposed directly. Internal routes often contain fields, behaviors, or operational actions that are not appropriate for external clients.

Company-scoped authorization must be visible in the contract. Each endpoint should explain how the token owner's company boundary constrains request handling and response data.

Support and security teams need predictable error and audit behavior. Clear contracts make it easier to investigate failures, scope mismatches, rate limits, and boundary denials.

Endpoint examples make the architecture reviewable without adding application code.

## Contract documentation template
- **Endpoint**: route template, such as `/external/v1/payments/{paymentId}`.
- **Method**: HTTP method.
- **Purpose**: external use case served by the endpoint.
- **Required scope**: scope needed before the operation can proceed.
- **Authentication**: credential requirement, such as `X-Api-Key`.
- **Company boundary**: company context resolved from token ownership.
- **Resource boundary**: resource ownership or visibility rule.
- **Idempotency**: idempotency key behavior where relevant.
- **Rate-limit expectation**: throttling expectations for the endpoint or operation type.
- **Audit expectation**: audit event expectations for allowed and denied requests.
- **Request parameters**: path and query parameters.
- **Request body**: accepted body shape, if any.
- **Successful response**: response shape and stable fields.
- **Error responses**: expected error codes and status mapping.
- **Security notes**: endpoint-specific security considerations.

## Common response conventions
- Use stable route templates.
- Include `requestId` in responses.
- Use consistent error codes.
- Avoid leaking internal implementation details.
- Use stable resource identifiers.
- Never return data outside the resolved token owner's company boundary.
- Avoid exposing internal enum names if they are not part of the external contract.

## Common error model
Conceptual error response fields:
- `error`: stable machine-readable error code.
- `message`: concise human-readable explanation.
- `requestId`: request reference for support and investigation.
- `details`: optional structured validation or conflict details.

Example error codes:
- `authentication_failed`
- `token_expired`
- `token_revoked`
- `insufficient_scope`
- `company_boundary_violation`
- `resource_not_found`
- `rate_limit_exceeded`
- `validation_failed`
- `idempotency_conflict`

Status mapping:
- Use `401` for authentication failures.
- Use `403` for authorization, scope, and company-boundary failures.
- Use `404` when a resource is not visible under the resolved company boundary.
- Use `409` for idempotency conflict.
- Use `422` for validation errors.
- Use `429` for throttling.
- `500`-class responses should not expose internal details.

Example:
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

## Error code reference
This document includes example endpoint error behavior. `docs/error-code-reference.md` is the canonical reference for stable client-facing error codes, HTTP status mapping, retry guidance, and audit expectations.

## Example contract: List banks
- **Endpoint**: `/external/v1/banks`
- **Method**: `GET`
- **Purpose**: list banks available to the resolved company.
- **Required scope**: `banks:read`
- **Authentication**: `X-Api-Key` required.
- **Company boundary**: resolved from token ownership.
- **Resource boundary**: only banks available to the resolved company.
- **Idempotency**: not required for read operation.
- **Rate-limit expectation**: moderate read limit by token, company, and endpoint.
- **Audit expectation**: audit allowed and denied access with token, company, endpoint, decision, reason, and request ID.

Request parameters:
| Name | Location | Required? | Notes |
|---|---|---|---|
| `countryCode` | query | no | Optional country filter using a stable external code. |
| `currencyCode` | query | no | Optional currency filter using a stable external code. |

Successful response:
```json
{
  "requestId": "req_banks_001",
  "banks": [
    {
      "bankId": "bank_001",
      "name": "Example Bank",
      "countryCode": "GB",
      "supportedCurrencies": ["GBP", "EUR"]
    }
  ]
}
```

Error cases:
- `authentication_failed`
- `insufficient_scope`
- `rate_limit_exceeded`

Security notes:
- The response must include only company-visible banks.
- Internal bank routing metadata should not be returned unless it is part of the external contract.

## Example contract: List accounts
- **Endpoint**: `/external/v1/accounts`
- **Method**: `GET`
- **Purpose**: list accounts linked to the resolved company.
- **Required scope**: `accounts:read`
- **Authentication**: `X-Api-Key` required.
- **Company boundary**: resolved from token ownership.
- **Resource boundary**: only accounts linked to the resolved company.
- **Idempotency**: not required for read operation.
- **Rate-limit expectation**: sustained read limit by token, company, and endpoint.
- **Audit expectation**: audit allowed and denied access with account-list resource type.

Request parameters:
| Name | Location | Required? | Notes |
|---|---|---|---|
| `bankId` | query | no | Optional filter. The bank must be visible to the resolved company. |

Successful response:
```json
{
  "requestId": "req_accounts_001",
  "accounts": [
    {
      "accountId": "acct_001",
      "bankId": "bank_001",
      "displayName": "Operating Account",
      "currency": "GBP",
      "status": "active"
    }
  ]
}
```

Error cases:
- `authentication_failed`
- `insufficient_scope`
- `resource_not_found`
- `rate_limit_exceeded`

Security notes:
- Do not return real IBANs, bank account numbers, or personal data.
- If `bankId` is not visible under the resolved company boundary, return `resource_not_found` rather than revealing another company's resource.

## Example contract: Create payment
- **Endpoint**: `/external/v1/payments`
- **Method**: `POST`
- **Purpose**: create a payment request under the resolved company.
- **Required scope**: `payments:create`
- **Authentication**: `X-Api-Key` required.
- **Company boundary**: resolved from token ownership.
- **Resource boundary**: payment created under resolved company only.
- **Idempotency**: `Idempotency-Key` header required.
- **Rate-limit expectation**: stricter write-operation limit by token, company, endpoint, and operation type.
- **Audit expectation**: sensitive payment action must be audited for allowed and denied requests.

Request headers:
| Name | Required? | Notes |
|---|---|---|
| `Idempotency-Key` | yes | Scoped to token, company, and endpoint. |

Request body:
| Field | Required? | Notes |
|---|---|---|
| `sourceAccountId` | yes | Must belong to the resolved company. |
| `beneficiaryReference` | yes | Stable external beneficiary reference, not personal data. |
| `amount` | yes | Positive decimal amount. |
| `currency` | yes | Supported currency code. |
| `executionDate` | yes | Requested execution date. |
| `clientReference` | no | Client-supplied reference for reconciliation. |

Example request body:
```json
{
  "sourceAccountId": "acct_001",
  "beneficiaryReference": "beneficiary_ref_001",
  "amount": "125.00",
  "currency": "GBP",
  "executionDate": "2026-05-15",
  "clientReference": "client_ref_001"
}
```

Successful response:
```json
{
  "requestId": "req_payment_create_001",
  "paymentId": "pay_001",
  "status": "accepted"
}
```

Error cases:
- `authentication_failed`
- `insufficient_scope`
- `validation_failed`
- `idempotency_conflict`
- `rate_limit_exceeded`

Security notes:
- Caller-provided `companyId` is ignored or rejected.
- Idempotency does not bypass authentication, scope, company boundary, or validation.
- Sensitive payment details should be minimized in responses and audit events.

## Example contract: Get payment status
- **Endpoint**: `/external/v1/payment-status/{paymentId}`
- **Method**: `GET`
- **Purpose**: retrieve status for a company-owned payment.
- **Required scope**: `payment-status:read`
- **Authentication**: `X-Api-Key` required.
- **Company boundary**: resolved from token ownership.
- **Resource boundary**: `paymentId` must belong to the resolved company.
- **Idempotency**: not required for read operation.
- **Rate-limit expectation**: polling should be rate-limited by token, company, endpoint, and payment resource.
- **Audit expectation**: audit allowed and denied status lookups with payment resource reference.

Request parameters:
| Name | Location | Required? | Notes |
|---|---|---|---|
| `paymentId` | path | yes | Must resolve under the token owner's company. |

Successful response:
```json
{
  "requestId": "req_payment_status_001",
  "paymentId": "pay_001",
  "status": "processing",
  "lastUpdatedAt": "2026-05-01T12:00:00Z"
}
```

Error cases:
- `authentication_failed`
- `insufficient_scope`
- `resource_not_found`
- `rate_limit_exceeded`

Security notes:
- Return `resource_not_found` when `paymentId` is outside the resolved company boundary.
- Clients should use backoff and avoid tight polling loops.

## Example contract: List notifications
- **Endpoint**: `/external/v1/notifications`
- **Method**: `GET`
- **Purpose**: list notifications for the resolved company.
- **Required scope**: `notifications:read`
- **Authentication**: `X-Api-Key` required.
- **Company boundary**: resolved from token ownership.
- **Resource boundary**: only notifications for the resolved company.
- **Idempotency**: not required for read operation.
- **Rate-limit expectation**: sustained read limit by token, company, and endpoint.
- **Audit expectation**: audit allowed and denied notification access.

Request parameters:
| Name | Location | Required? | Notes |
|---|---|---|---|
| `since` | query | no | Return notifications after this timestamp. |
| `limit` | query | no | Maximum number of notifications to return, within documented bounds. |
| `type` | query | no | Optional notification type filter. |

Successful response:
```json
{
  "requestId": "req_notifications_001",
  "notifications": [
    {
      "notificationId": "notif_001",
      "type": "payment_status_changed",
      "resourceId": "pay_001",
      "occurredAt": "2026-05-01T12:00:00Z"
    }
  ]
}
```

Error cases:
- `authentication_failed`
- `insufficient_scope`
- `validation_failed`
- `rate_limit_exceeded`

Security notes:
- Notification contents should not expose data outside the resolved company boundary.
- Keep notification payloads minimal and stable.

## Example contract: Register webhook
- **Endpoint**: `/external/v1/webhooks`
- **Method**: `POST`
- **Purpose**: register a webhook endpoint for the resolved company.
- **Required scope**: `webhooks:manage`
- **Authentication**: `X-Api-Key` required.
- **Company boundary**: resolved from token ownership.
- **Resource boundary**: webhook registered for resolved company.
- **Idempotency**: recommended for repeated registration attempts.
- **Rate-limit expectation**: strict webhook management limit.
- **Audit expectation**: webhook management action must be audited.

Request body:
| Field | Required? | Notes |
|---|---|---|
| `url` | yes | HTTPS endpoint controlled by the client. |
| `eventTypes` | yes | External event types to deliver. |
| `description` | no | Human-readable description for operators. |

Example request body:
```json
{
  "url": "https://partner.example.com/webhooks/payments",
  "eventTypes": ["payment_status_changed"],
  "description": "Payment updates"
}
```

Successful response:
```json
{
  "requestId": "req_webhook_001",
  "webhookId": "wh_001",
  "status": "active"
}
```

Error cases:
- `authentication_failed`
- `insufficient_scope`
- `validation_failed`
- `rate_limit_exceeded`

Security notes:
- Webhook URL validation should be explicit.
- Internal callbacks or private network targets should not be allowed unless explicitly controlled by platform policy.
- Webhook management actions should remain company-scoped.

## Webhook delivery
Webhook registration is only one part of the model. Actual delivery should use signed, company-scoped, minimal event payloads with bounded retries and auditable outcomes.

Event schemas should be documented as part of the external product surface. Clients should not rely on internal event names or undocumented payload fields.

## Idempotency guidance
Write operations should support idempotency. Payment creation should require `Idempotency-Key`.

Idempotency keys should be scoped to token, company, and endpoint. Conflicting reused keys should return stable conflict behavior.

Idempotency should not bypass authentication, scope, company boundary, or validation.

## Versioning guidance
Use versioned external routes such as `/external/v1`.

Internal service versions should not leak into the external contract. Breaking changes require a new version or migration policy.

Additive changes should be documented clearly. Deprecated fields should have a clear sunset path.

## Common mistakes
- Exposing internal endpoints directly.
- Relying on internal Swagger as external documentation.
- Omitting required scope from endpoint docs.
- Not documenting company/resource boundary.
- Returning resources from other companies.
- Using inconsistent error codes.
- Exposing internal exception messages.
- Skipping idempotency for write operations.
- Not documenting audit expectations.
- Treating request `companyId` as authoritative.

## Implementation notes
This model is language-agnostic and conceptual. It does not define application code, framework boilerplate, or an OpenAPI/YAML file.

Teams may later translate these contract examples into OpenAPI, API portal documentation, or integration guides. The important requirement is that external contracts remain explicit, stable, and separated from internal API surfaces.
