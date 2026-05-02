# External API Rate Limiting

This document describes an illustrative rate limiting and abuse detection model for company-scoped external API access in a fintech-style reference architecture. It is a non-production reference and stays at the design level: it does not prescribe algorithms, gateway products, cloud services, queues, databases, monitoring products, or exact thresholds.

Rate limiting is a design-level control for reducing excessive usage, noisy integrations, retry loops, and abuse signals. Successful authentication and scope authorization do not guarantee request execution; a request can still be limited or denied before product or platform data is accessed.

## Related Documents

- [Request flow diagrams](../architecture/request-flow.md)
- [API key lifecycle](../token-lifecycle/api-key-lifecycle.md)
- [Permission model](../scopes/permission-model.md)
- [External API audit log](../auditability/external-api-audit-log.md)
- [Threat model](../threat-model.md)
- [Internal vs external API](../integration-boundaries/internal-vs-external-api.md)
- [Error code reference](../error-code-reference.md)
- [Minimal illustrative OpenAPI contract](../../examples/openapi/external-api.sample.yaml)

## Why Rate Limiting Matters

External API clients can create operational risk even when they use valid API keys and valid scopes. Examples include repeated polling, retry loops, high write volume, repeated invalid-token attempts, and attempts to access resources outside the resolved company scope.

Rate limiting and abuse detection help:

- reduce excessive external API load,
- isolate noisy tokens or integrations,
- protect company-scoped capacity,
- control expensive or sensitive endpoints,
- make retry behavior predictable for clients,
- provide audit and monitoring signals for investigation.

This model should be read as security-focused design notes, not as a claim of complete enforcement.

## Request-Time Placement

Rate limiting can be evaluated after token validation, company resolution, and scope checks, because those steps provide trusted context:

1. Validate the presented `X-Api-Key`.
2. Load token status, owner, expiry, and scopes.
3. Resolve effective `CompanyId` from the token owner.
4. Check the endpoint's required scope.
5. Evaluate rate-limit and abuse-control policies.
6. Allow the operation or return a stable denial response.
7. Audit the allowed, rate-limited, or abuse-denied decision.

The [rate-limit request flow](../architecture/request-flow.md#rate-limit-or-abuse-detection-denial) shows this ordering. Credential-denied requests can also feed abuse signals even when no company scope is resolved.

## Limit Dimensions

Rate limits can be conceptualized across several dimensions. A single global limit is rarely enough for company-scoped external API access.

| Dimension | Purpose | Generic example |
|---|---|---|
| Token | Isolate one noisy or misused API key. | A single integration repeatedly calls account-read endpoints. |
| Company | Control aggregate usage across all keys owned by one company. | Multiple tokens owned by one company create high read volume. |
| Endpoint | Protect routes with different operational cost or sensitivity. | Payment initiation requests are limited differently from account reads. |
| Scope or capability | Apply policy by external capability rather than only route path. | `payments:create` has stricter controls than `accounts:read`. |
| Operation type | Distinguish reads, writes, administration, and webhook management. | Token administration and webhook changes are monitored separately. |
| Source pattern | Support investigation of unusual caller patterns. | Repeated invalid requests arrive from changing source addresses. |

These dimensions can be combined at design level. For example, an external payment initiation request can be evaluated by token, company, endpoint, write-operation category, and `payments:create` capability.

## Example Limit Dimensions

Concise examples using generic fintech operations:

- account read requests by token,
- transaction read requests by company,
- payment initiation requests by token and company,
- webhook management requests by company,
- token administration requests by token owner,
- repeated credential failures by key prefix or source pattern,
- missing-scope denials by endpoint and required scope.

The exact policy values should be chosen by the implementation and operating context. This reference only describes the design shape.

## Abuse Detection Signals

Abuse detection can identify suspicious patterns beyond simple request counts.

Example signals:

- repeated missing, unknown, invalid, expired, or revoked API key attempts,
- repeated requests with missing required scope,
- attempts to provide or override company scope,
- repeated resource access outside the resolved company scope,
- high write-operation volume,
- aggressive polling of account, transaction, or payment status endpoints,
- repeated retries without backoff after `429` responses,
- unusual source address or user-agent changes for the same token,
- calls to unknown, internal-looking, or undocumented routes,
- rapid webhook management changes,
- token administration attempts outside expected patterns.

Abuse signals can lead to softer monitoring actions, stricter temporary throttling, deactivation review, or token revocation when token misuse is suspected. These are separate controls and should be represented clearly.

## Temporary Throttling vs Token Revocation

Temporary throttling and token revocation are different controls.

Temporary throttling:

- responds to current request volume or suspicious traffic patterns,
- can return `429 Too Many Requests` or another safe denial response,
- may change as traffic patterns normalize,
- should be auditable as a rate-limit or abuse-control decision.

Token revocation:

- permanently disables a token before its natural expiry,
- is a token lifecycle action,
- should be used when an access grant should no longer be usable,
- should be auditable as a lifecycle event and should cause future requests with that token to fail authentication.

Rate limiting should not be used as a substitute for revoking a token that should no longer be trusted. Revocation should not be used merely to manage ordinary short-term traffic spikes.

## Denial Response Behavior

Rate-limit and abuse-control denials should produce stable external error behavior.

For hard rate limits, a `429 Too Many Requests` response can include a safe error code such as `rate_limit_exceeded`, a correlation ID, and retry guidance when safe. The response should not reveal exact internal thresholds, capacity, queue depth, policy internals, database state, or other companies' usage.

Abuse-control denials should also use documented, safe response behavior. Depending on the external contract, they can reuse stable rate-limit or forbidden-style semantics without exposing detection details.

See the [error code reference](../error-code-reference.md) and [OpenAPI sample](../../examples/openapi/external-api.sample.yaml) for stable external error-shape examples.

## Auditability

Rate-limit and abuse-control denials should be auditable without storing raw API keys or sensitive payloads.

Audit events should capture safe metadata such as:

- event type,
- decision result such as `rate_limited` or `denied`,
- error code or error classification,
- HTTP status,
- correlation ID,
- token identifier or key prefix when available,
- token owner company when available,
- effective `CompanyId` when resolved,
- route template,
- required scope or capability when relevant,
- policy identifier or category when safe,
- operation type,
- client IP or user-agent where useful,
- timestamp.

Audit records should not include raw API keys, token hashes, full sensitive request bodies, full sensitive response bodies, exact internal thresholds, or sensitive operational state.

See the [external API audit log](../auditability/external-api-audit-log.md) for the broader audit event model.

## Interaction with Scopes and Token Lifecycle

Rate limiting does not replace authentication, token lifecycle checks, or endpoint-level authorization.

- A missing, unknown, expired, revoked, or deactivated key should be denied by token lifecycle controls before normal scoped execution.
- A valid token without the required scope should be denied by the permission model before product data access.
- A valid, scoped request can still be rate-limited or abuse-denied.
- Repeated lifecycle or scope denials can feed abuse detection signals.

This ordering keeps the platform from treating request volume as permission and keeps token revocation distinct from throttling.

## Operational Review Questions

Rate-limit and abuse policies should be reviewed when:

- new external endpoints are added,
- new write operations are introduced,
- scope requirements change,
- an integration's normal traffic pattern changes,
- audit logs show repeated credential failures or missing-scope denials,
- clients report repeated `429` responses,
- external traffic reaches unknown or internal-looking routes,
- token misuse or replay is suspected.

Policy changes should be documented at the design level and aligned with the external contract, audit model, and token lifecycle expectations.

## Common Mistakes

- Treating successful authentication as permission to execute unlimited requests.
- Applying only a global limit with no token or company context.
- Keying limits only by source address.
- Using rate limits to hide missing authorization checks.
- Returning unstable or overly detailed rate-limit errors.
- Exposing internal capacity or policy thresholds to clients.
- Failing to audit rate-limit and abuse-control denials.
- Storing raw API keys or sensitive payloads in rate-limit diagnostics.
- Revoking tokens for ordinary temporary traffic spikes.
- Using throttling instead of revocation when a token should no longer be trusted.

## Design-Level Review Checklist

- [ ] Are rate-limit checks placed after token validation, company resolution, and scope checks when trusted context is needed?
- [ ] Are token, company, endpoint, and scope/capability dimensions considered?
- [ ] Are sensitive write operations treated differently from low-risk reads where useful?
- [ ] Are repeated credential, scope, and company-boundary denials available as abuse signals?
- [ ] Do rate-limit and abuse-control denials return stable external errors?
- [ ] Are denials auditable with token, company, endpoint, decision, and correlation metadata when available?
- [ ] Are raw API keys, token hashes, sensitive payloads, and internal capacity details excluded?
- [ ] Are temporary throttling and token revocation documented as separate controls?
