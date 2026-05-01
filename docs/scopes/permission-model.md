# Permission Model

External API keys should use explicit endpoint-level scopes. A scope describes what an external client is allowed to do through the external API contract.

Scopes are not internal role names. They are stable external permissions that can be documented, reviewed, granted, denied, and audited.

## Why Scopes Exist

Scopes reduce the blast radius of an API key. A client that only needs to read available banks should not receive payment creation access. A webhook management integration should not automatically receive account read access.

Scopes support:

- least privilege,
- endpoint-level authorization,
- safer onboarding,
- clearer support review,
- auditability of allowed and denied requests,
- controlled rollout of new external API capabilities.

## Read vs Write Scopes

Read scopes authorize retrieval of data. Write scopes authorize state-changing operations.

Read and write scopes should be separated even when they apply to the same domain. For example, `payments:read` allows payment status retrieval, while `payments:create` allows a client to initiate a payment request.

Write scopes generally require more operational review because they can create new records, trigger workflows, or change customer-facing state.

## Endpoint-Level Access

Each external endpoint should declare the scope required to call it. Authorization should happen before application services return data or perform mutations.

Endpoint-level scopes are easier to review than broad roles because each route can be checked against a clear permission requirement.

## Company-Level Isolation

Scopes authorize actions only inside the company scope resolved from the API key. A token with `accounts:read` can read accounts only for the owning company.

A scope must never make a token global. Company isolation is resolved before endpoint authorization and remains attached to the request context throughout the operation.

## Deny-by-Default Behavior

Authorization should deny requests when:

- the API key is missing or invalid,
- the key is revoked, expired, or deactivated,
- the endpoint requires a scope the key does not have,
- the endpoint is unknown,
- the request attempts to override company scope,
- the requested resource is outside the resolved company scope.

Denied requests should be recorded in audit logs with a stable decision result and error classification.

## Scope Naming Convention

Use simple `resource:action` names.

Examples:

- `banks:read`
- `accounts:read`
- `payments:read`
- `payments:create`
- `webhooks:manage`
- `tokens:manage`

Guidelines:

- use lowercase names,
- keep resources plural,
- use verbs that match the external API contract,
- avoid internal team, service, or role names,
- avoid wildcard scopes until there is a clear operational need and review process.

## Sample Scope-to-Endpoint Table

| Method | Endpoint | Required scope | Notes |
|---|---|---|---|
| `GET` | `/external/v1/banks` | `banks:read` | Lists supported banks for the resolved company scope. |
| `GET` | `/external/v1/accounts` | `accounts:read` | Lists accounts visible to the resolved company scope. |
| `GET` | `/external/v1/payments/{payment_id}` | `payments:read` | Reads one payment if it belongs to the resolved company scope. |
| `POST` | `/external/v1/payments` | `payments:create` | Creates a payment request inside the resolved company scope. |
| `GET` | `/external/v1/webhooks` | `webhooks:manage` | Lists configured webhook endpoints. |
| `POST` | `/external/v1/webhooks` | `webhooks:manage` | Creates a webhook endpoint configuration. |
| `POST` | `/external/v1/tokens/{token_id}/revoke` | `tokens:manage` | Revokes an API key owned by the resolved company scope. |

## Common Mistakes

### Using Role Names as External Scopes

Internal roles such as `admin`, `operator`, or `support` are usually too broad and too tied to internal workflows. External scopes should describe API capabilities, not employee permissions.

### Trusting Client-Provided Company Identifiers

A token's company scope must come from platform-owned API key metadata. Client-provided company identifiers can be useful as business filters only after the platform has resolved the trusted company scope.

### Making Tokens Global

Global tokens increase blast radius and make audit review harder. External API keys should belong to a company and should authorize only that company's data and operations.

### Allowing Wildcard Permissions Too Early

Wildcard permissions such as `*:*` or `payments:*` are convenient but risky. They can hide newly added endpoint access and weaken review discipline. Prefer explicit scopes until a mature governance model exists.

### Not Auditing Denied Requests

Denied requests are security signals. Missing scope decisions, company-scope violations, revoked token usage, and rate-limited requests should be captured for investigation and operational monitoring.
