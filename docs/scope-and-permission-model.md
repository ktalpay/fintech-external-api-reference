# Scope and permission model

## Purpose
External API keys need both tenant/company ownership boundaries and endpoint-level operation permissions.

Company ownership establishes the tenant boundary for every request. It prevents one external client from reading or modifying another company's resources, even if the request includes a different `companyId` in a path, query string, or payload.

Endpoint-level operation permissions establish what the token is allowed to do inside that tenant boundary. A token may be allowed to read bank metadata without being allowed to create payments, manage webhooks, or read notifications.

Together, these controls prevent cross-tenant access and reduce the blast radius of a leaked, overused, or misconfigured token.

## Design goals
- **Tenant isolation**: every request is evaluated under the company that owns the token.
- **Least privilege**: tokens receive only the scopes needed for the intended integration.
- **Deny-by-default access**: missing, unknown, expired, revoked, or underscoped access is denied.
- **Explicit endpoint contracts**: every external endpoint declares the scope required to call it.
- **Auditable permission decisions**: authorization outcomes are traceable to token, company, endpoint, scope, and decision.
- **Separation between external scopes and internal application roles**: external API permissions are stable integration contracts, not direct copies of internal roles.

## Non-goals
- Replacing internal IAM.
- Modelling every internal role.
- Implementing OAuth/OIDC.
- Defining legal/compliance obligations.
- Providing production-ready code.

## Core concepts
- **Company ownership**: the tenant relationship recorded when a token is issued. The token belongs to exactly one company.
- **API key identity**: the non-secret token record resolved from the presented API key, including token ID, fingerprint, status, expiry, owning company, and scopes.
- **Scope**: a named external permission granted to a token, expressed as a stable integration contract such as `payments:create`.
- **Permission**: the authorization decision made by comparing token state, token scopes, endpoint requirements, and resource ownership.
- **Resource boundary**: the company-scoped limit for data access, such as accounts, payments, notifications, or webhooks owned by the token owner's company.
- **Endpoint contract**: the documented method, path, required scope, resource boundary, and expected authorization behavior for an external API endpoint.
- **Product boundary**: the separation between the external API layer and internal product/domain services. External scopes should not expose or mirror internal role models.

## Recommended scope naming convention
Use a simple `resource:action` format.

Examples:
- `banks:read`
- `accounts:read`
- `payments:create`
- `payments:read`
- `payment-status:read`
- `notifications:read`
- `webhooks:manage`

Broad wildcard scopes should be avoided in early versions because they obscure intent, weaken least privilege, and make audits less useful. It is easier to add a broader scope later than to safely narrow a widely issued token after integrations depend on it.

## Scope evaluation rules
1. Token must be active.
2. Token must not be expired.
3. Token must belong to exactly one company.
4. Endpoint must require an explicit scope.
5. Token must contain the required scope.
6. Request must be evaluated under the token owner's `CompanyId`.
7. Request payload `CompanyId` must not override token ownership.
8. Denied checks should be audited.
9. Sensitive allowed operations should be audited.

## Endpoint permission matrix
| Endpoint | Method | Required scope | Resource boundary | Notes |
|---|---|---|---|---|
| /external/v1/banks | GET | banks:read | Token owner's CompanyId | Return only company-visible banks |
| /external/v1/accounts | GET | accounts:read | Token owner's CompanyId | Return only accounts linked to the company |
| /external/v1/payments | POST | payments:create | Token owner's CompanyId | Create payment request under resolved company |
| /external/v1/payments/{paymentId} | GET | payments:read | Token owner's CompanyId + payment ownership | Payment must belong to resolved company |
| /external/v1/payment-status/{paymentId} | GET | payment-status:read | Token owner's CompanyId + payment ownership | Status lookup must not cross tenant boundary |
| /external/v1/notifications | GET | notifications:read | Token owner's CompanyId | Return company-scoped notifications |
| /external/v1/webhooks | POST | webhooks:manage | Token owner's CompanyId | Register webhook for resolved company |
| /external/v1/webhooks/{webhookId} | DELETE | webhooks:manage | Token owner's CompanyId + webhook ownership | Delete only company-owned webhook |

## Deny-by-default behavior
- Unknown endpoint => deny.
- Endpoint without declared required scope => deny.
- Token without matching scope => deny.
- Company mismatch => deny.
- Expired/revoked token => deny.

## Resource-level filtering
- Bank list results must be filtered by the token owner's `CompanyId`.
- Accounts must be filtered by the token owner's `CompanyId`.
- Payment IDs must be resolved under the token owner's `CompanyId`.
- Notifications must be filtered by the token owner's `CompanyId`.
- Webhook IDs must be resolved under the token owner's `CompanyId`.

## Product/domain service boundary
The external API layer performs token lookup, scope evaluation, and company resolution before calling product/domain services.

Product services still enforce company-scoped queries. They should receive a platform-resolved company context and apply that context to resource lookups rather than trusting external request fields.

Downstream services must not trust unvalidated external payloads. Internal roles should not be directly reused as external scopes because external scopes describe partner-facing API contracts, while internal roles describe application or staff permissions.

## Operational safeguards
- Audit failed permission checks.
- Audit successful access to sensitive operations.
- Monitor repeated denied requests.
- Separate read and write scopes.
- Avoid wildcard scopes in early versions.
- Rotate or revoke suspicious tokens.
- Track last-used metadata.

## Example request evaluation scenarios
### Valid banks:read access
- **Input**: active, unexpired token owned by `companyId = 12` with `banks:read`; request calls `GET /external/v1/banks`.
- **Evaluation**: token is valid, endpoint requires `banks:read`, token contains the scope, and results are filtered under company `12`.
- **Result**: allow.
- **Audit event expectation**: record successful bank-list access with token ID/fingerprint, company ID, endpoint, required scope, and allow decision.

### Missing payments:create scope
- **Input**: active, unexpired token owned by `companyId = 12` with `payments:read`; request calls `POST /external/v1/payments`.
- **Evaluation**: token is valid, endpoint requires `payments:create`, and token does not contain the required scope.
- **Result**: deny.
- **Audit event expectation**: record denied access with missing-scope reason, token ID/fingerprint, company ID, endpoint, required scope, and deny decision.

### Request payload attempts to override CompanyId
- **Input**: active, unexpired token owned by `companyId = 12` with `payments:create`; request payload includes `companyId = 99`.
- **Evaluation**: token company remains authoritative. The payload value cannot override token ownership and must be rejected or ignored according to the endpoint contract.
- **Result**: deny if the payload company conflicts with token ownership; otherwise process under company `12`.
- **Audit event expectation**: record denied company-mismatch access when a conflicting company value is present.

### Expired token
- **Input**: token owned by `companyId = 12` with `accounts:read`, but expiry timestamp is in the past; request calls `GET /external/v1/accounts`.
- **Evaluation**: expiry check fails before endpoint authorization is allowed.
- **Result**: deny.
- **Audit event expectation**: record denied access with expired-token reason, token ID/fingerprint when available, endpoint, and deny decision.

### Token revoked after suspected leakage
- **Input**: token owned by `companyId = 12` with `webhooks:manage`, but token status is revoked after suspected leakage; request calls `POST /external/v1/webhooks`.
- **Evaluation**: status check fails before scope authorization.
- **Result**: deny.
- **Audit event expectation**: record denied access with revoked-token reason, token ID/fingerprint, company ID, endpoint, and deny decision.

## Common mistakes
- Using only a boolean `IsActive` check.
- Trusting `CompanyId` from request body.
- Reusing internal roles as external scopes.
- Giving one token access to every endpoint.
- Not logging denied access attempts.
- Filtering after data is already loaded too broadly.
- Letting product services infer company context from external input.

## Implementation notes
This model is language-agnostic and conceptual. It does not define application code.

Implementation may use policy middleware, endpoint metadata, or declarative route configuration depending on the platform. The important requirement is that token state, required scope, resolved company ownership, resource boundary, and audit behavior are consistently enforced for every external endpoint.
