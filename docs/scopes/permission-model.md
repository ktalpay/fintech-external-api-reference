# Permission Model

This document describes an illustrative permission model for company-scoped external API keys in a fintech-style reference architecture. It is a non-production reference and stays at the design level: it does not define a full RBAC or ABAC framework.

Scopes define which external API capabilities a token may use. They do not define company ownership, and they must not override the company scope resolved from API key metadata.

## Related Documents

- [Request flow diagrams](../architecture/request-flow.md)
- [API key lifecycle](../token-lifecycle/api-key-lifecycle.md)
- [Threat model](../threat-model.md)
- [External API audit log](../auditability/external-api-audit-log.md)
- [External API rate limiting](../rate-limiting/external-api-rate-limiting.md)
- [Internal vs external API](../integration-boundaries/internal-vs-external-api.md)
- [Minimal illustrative OpenAPI contract](../../examples/openapi/external-api.sample.yaml)

## Ownership vs Permission

External API authorization has two separate questions:

1. Which company owns this token?
2. Which external API capabilities does this token allow inside that company scope?

The ownership rule is:

1. `X-Api-Key` resolves to a token record.
2. The token record has an owning company.
3. The owning company determines the effective `CompanyId`.
4. External callers must not choose arbitrary company scope.

Scopes answer the second question only. For example, `accounts:read` allows account-read capability only inside the company scope resolved from the token owner. It does not make the token global, and it does not let the caller provide another `CompanyId` to expand access.

## Request-Time Order

Permission checks happen after token validation and company resolution:

1. Validate the presented `X-Api-Key`.
2. Load token metadata, including status, owner, expiry, and scopes.
3. Deny missing, unknown, expired, revoked, or deactivated keys.
4. Resolve effective `CompanyId` from the token owner.
5. Match the endpoint to its required scope.
6. Deny if the token lacks the required scope.
7. Apply rate-limit and abuse controls.
8. Execute the operation inside the resolved company scope.
9. Audit the allowed or denied decision.

See the [request flow diagrams](../architecture/request-flow.md) for the successful flow and the scope-denial path.

## Why Scopes Exist

Scopes reduce the impact of token misuse and make external access reviewable. A token used only for account reads should not automatically be able to initiate payment requests or manage webhooks.

Scopes support:

- least privilege,
- endpoint-level authorization,
- safer token creation and rotation,
- auditable allowed and denied decisions,
- clearer external API documentation,
- controlled rollout of new external capabilities.

## Example Scope Categories

Use simple `resource:action` names that describe external API capabilities.

| Category | Example scope | External capability |
|---|---|---|
| Account read | `accounts:read` | List or read accounts belonging to the resolved company scope. |
| Transaction read | `transactions:read` | Read transaction data visible inside the resolved company scope. |
| Payment initiation request | `payments:create` | Create a payment initiation request inside the resolved company scope. |
| Payment read | `payments:read` | Read payment request status for resources inside the resolved company scope. |
| Webhook management | `webhooks:manage` | Create, list, update, or remove webhook configurations for the resolved company. |
| Token administration | `tokens:manage` | Rotate or revoke external API keys owned by the resolved company. |

These examples are illustrative. A real implementation can split broad categories into narrower scopes when review, risk, or client needs require it.

## Read and Write Separation

Read scopes authorize retrieval of data. Write scopes authorize state-changing operations.

Read and write scopes should usually be separate even when they apply to the same domain. For example, `payments:read` can allow payment status retrieval, while `payments:create` can allow a payment initiation request.

Write scopes generally deserve closer review because they can create records, trigger workflows, or change client-visible state.

## Endpoint-Level Access

Each external endpoint should declare the scope required to call it. Authorization should happen before product or platform data is returned or mutated.

Endpoint-level scopes are easier to review than broad roles because each route can be checked against a clear permission requirement. They also make external contract examples easier to read: the required scope can be documented next to the route, expected error responses, and rate-limit behavior.

## Company-Level Isolation

Scopes authorize actions only inside the company scope resolved from the token owner.

A scope must never:

- make a token global,
- allow the caller to select another company,
- bypass resource ownership checks,
- substitute for token lifecycle validation,
- expose internal-only endpoints.

Client-provided company identifiers can be filters or resource references only after the platform has resolved trusted company scope. Product or platform data access should still enforce that the requested resource belongs to the resolved company scope.

## Deny-by-Default Behavior

Authorization should deny requests when:

- the API key is missing or invalid,
- the key is expired, revoked, or deactivated,
- the endpoint requires a scope the token does not have,
- the endpoint is unknown or not part of the external contract,
- the request attempts to override company scope,
- the requested resource is outside the resolved company scope.

Scope-denied requests should produce stable external error behavior, such as a documented `403 Forbidden` response with a safe error code and correlation ID. Client-facing messages should not expose internal service names, stack traces, token hashes, raw token values, or sensitive operational details.

Denied scope checks should be audited with a stable decision result and error classification.

## Scope Changes and Auditability

Scope grants and removals affect what a token can do and should be auditable lifecycle changes.

Audit records for scope changes should include safe metadata such as:

- token identifier or key prefix,
- owning company,
- added or removed scopes,
- actor or initiating system where available,
- timestamp,
- reason or change reference where appropriate,
- correlation ID where useful.

Request-time audit records should capture missing-scope denials, company-scope violations, and successful use of sensitive scopes. Audit records should not include raw API keys, token hashes, full sensitive request bodies, or full sensitive response bodies.

## Over-Broad Scope Risk

Over-broad scopes are a design risk because they increase the impact of token leakage, replay, or client-side misuse.

Examples of risky patterns:

- granting write scopes to read-only integrations,
- using wildcard scopes before there is a clear review process,
- bundling unrelated capabilities into one broad scope,
- granting token administration with unrelated data access,
- allowing a scope to imply access to future endpoints without review.

Prefer explicit scopes until there is a clear operational need for broader grouping. When broader scopes are introduced, document the behavior, review the affected endpoints, and audit token changes that grant the broader capability.

## Scope Naming Guidelines

Use names that are stable, external, and easy to review.

Guidelines:

- use lowercase `resource:action` names,
- keep resources plural,
- use verbs that match the external API contract,
- separate read and write actions,
- avoid internal team, service, or role names,
- avoid caller-selected company scope in the scope name,
- avoid wildcard scopes until review and audit expectations are clear.

## Sample Scope-to-Endpoint Table

| Method | Endpoint | Required scope | Notes |
|---|---|---|---|
| `GET` | `/external/v1/accounts` | `accounts:read` | Lists accounts inside the resolved company scope. |
| `GET` | `/external/v1/transactions` | `transactions:read` | Lists transactions visible inside the resolved company scope. |
| `GET` | `/external/v1/payments/{payment_id}` | `payments:read` | Reads one payment request if it belongs to the resolved company scope. |
| `POST` | `/external/v1/payments` | `payments:create` | Creates a payment initiation request inside the resolved company scope. |
| `GET` | `/external/v1/webhooks` | `webhooks:manage` | Lists configured webhook endpoints for the resolved company. |
| `POST` | `/external/v1/webhooks` | `webhooks:manage` | Creates a webhook endpoint configuration for the resolved company. |
| `POST` | `/external/v1/tokens/{token_id}/revoke` | `tokens:manage` | Revokes a token owned by the resolved company scope. |

The [OpenAPI sample](../../examples/openapi/external-api.sample.yaml) provides a minimal external contract shape with API key authentication and stable `401`, `403`, and `429` responses.

## Common Mistakes

### Treating Scopes as Ownership

Scopes do not decide tenant ownership. The token owner determines effective `CompanyId`; scopes only decide allowed capabilities inside that scope.

### Trusting Client-Provided Company Identifiers

A request body, query parameter, or header should not be trusted to expand company access. Caller-provided identifiers can be interpreted only after company scope has been resolved from token metadata.

### Using Internal Roles as External Scopes

Internal roles such as `admin`, `operator`, or `support` are usually too broad and too tied to internal workflows. External scopes should describe external API capabilities, not employee permissions.

### Making Tokens Global

Global tokens increase blast radius and make audit review harder. External API keys should belong to one company and authorize only that company's data and operations.

### Not Auditing Denied Requests

Denied requests are useful security signals. Missing-scope decisions, company-scope violations, revoked token usage, and rate-limited requests should be captured for investigation and operational monitoring.

## Design-Level Review Checklist

- [ ] Does each external endpoint declare one clear required scope?
- [ ] Are company ownership and permission scopes documented as separate concepts?
- [ ] Is effective `CompanyId` resolved from token owner metadata before scope checks?
- [ ] Are caller-provided company identifiers treated as untrusted for authorization?
- [ ] Are read and write scopes separated where useful?
- [ ] Are broad or wildcard scopes avoided unless there is a clear review process?
- [ ] Are scope grants, removals, and missing-scope denials auditable?
- [ ] Do scope-denied responses use stable external error behavior?
