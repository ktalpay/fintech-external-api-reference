# Threat Model

## Purpose

This illustrative threat model reviews the external API boundary for a company-scoped fintech-style reference architecture. It focuses on API key authentication, token ownership, company scope resolution, endpoint-level scopes, rate limiting, auditability, external contract isolation, and safe error responses.

This is a non-production reference and a set of security-focused design notes. It is not a complete security review, operational runbook, legal review, or regulatory assessment.

## Scope

In scope:

- external API endpoints under `/external/v1`,
- API key authentication via `X-Api-Key`,
- company-scoped authorization,
- endpoint-level scope and permission evaluation,
- audit logging for allowed and denied decisions,
- rate limiting and abuse detection,
- token lifecycle checks for active, expired, revoked, and deactivated keys,
- external API contract behavior and error response shape.

Out of scope:

- full cloud infrastructure threat modelling,
- full identity and access management program design,
- fraud detection,
- client-side application security,
- detailed incident response procedures,
- runtime implementation details.

## Company-Scoped Ownership Rule

External clients authenticate with `X-Api-Key`. The platform validates the presented key against the API key/token store, then resolves the token owner from trusted token metadata.

The token owner determines the effective company scope. Callers must not provide arbitrary `CompanyId`, `company_id`, or `X-Company-Id` values to expand access. Client-provided identifiers can be request input only after the platform has resolved trusted company scope from token metadata.

See the [request flow diagrams](architecture/request-flow.md) and [external API access model](architecture/external-api-access-model.md) for the order of authentication, company resolution, scope checks, rate limiting, data access, audit logging, and denial paths.

## Assets

| Asset | Why it matters | Protection expectation |
|---|---|---|
| Raw API key | Grants machine-to-machine access when presented by a caller. | Display once at creation; never store or log. |
| Token hash | Verifies a presented token without storing the raw secret. | Store as an authentication artifact; do not expose in logs or audit records. |
| Token metadata | Carries owner, status, expiry, scopes, key prefix, and lifecycle timestamps. | Treat as authoritative platform-controlled authorization context. |
| Company ownership mapping | Defines the tenant context for external requests. | Resolve from token metadata only. |
| External API contract | Defines the documented external surface and error responses. | Keep separate from internal APIs and review intentionally. |
| Product or platform data | Includes company-scoped resources such as accounts, banks, or payment requests. | Access only through resolved company scope and required endpoint scopes. |
| Audit events | Explain allowed, denied, lifecycle, and rate-limit decisions. | Keep append-only, queryable, and free of raw secrets. |
| Rate-limit counters | Support fair use and abuse response. | Key by token, company, endpoint, and operation type where appropriate. |
| Correlation IDs | Connect client-facing responses to operational records. | Propagate consistently without embedding sensitive data. |

## Trust Boundaries

| Boundary | What crosses it | Main risk | Design-level control |
|---|---|---|---|
| External client to External Integration API | Request headers, `X-Api-Key`, route, query parameters, and body. | Spoofed identity, malformed input, credential misuse, request flood. | Validate tokens, reject untrusted company scope, apply stable error responses and rate limits. |
| External Integration API to API key/token store | Key prefix, presented token verification, token metadata lookup. | Incorrect token resolution or stale lifecycle status. | Store only token hashes and check active, expired, revoked, and deactivated status at request time. |
| External Integration API to scope evaluator | Resolved token metadata, required endpoint scope, request context. | Over-broad or missing authorization. | Use endpoint-level scopes and deny by default. |
| External Integration API to rate-limit and abuse controls | Token, company, endpoint, operation, IP, and denial patterns. | Unfair throttling, bypass, or noisy integrations. | Use layered per-token, per-company, endpoint, and abuse-signal controls. |
| External Integration API to product or platform data boundary | Resolved company scope, validated request, resource references. | Cross-company data access or downstream trust in caller input. | Pass platform-resolved company context into scoped product queries. |
| External Integration API to audit log | Decision result, token identifier, resolved company scope, endpoint, correlation ID. | Missing traceability or sensitive data in audit records. | Record allowed and denied decisions with data minimization. |
| Internal services to external contract publication | Endpoint paths, schemas, examples, error shapes. | Internal behavior exposed as an external contract. | Keep internal APIs separate from curated external OpenAPI contracts. |

## Threat Entries

### API Key Leakage

**Threat:** A raw API key is copied from a client environment, support transcript, log line, browser history, or local configuration file.

**Impact:** An actor with the raw key can attempt external API access under the token owner's company scope until the key is rotated, revoked, expired, or otherwise denied.

**Design-level mitigation:** Display raw keys once, store only token hashes, keep raw tokens out of logs and audit records, support rotation and revocation, and record suspicious usage by token identifier or key prefix.

**Related docs:** [API key lifecycle](token-lifecycle/api-key-lifecycle.md), [external API audit log](auditability/external-api-audit-log.md), [request flow diagrams](architecture/request-flow.md).

### Stolen or Replayed API Keys

**Threat:** A valid API key is reused from an unexpected system, replayed by a script, or sent repeatedly from unusual source patterns.

**Impact:** Requests may succeed within the token owner's company scope and granted scopes, creating unauthorized reads or state-changing operations for that company.

**Design-level mitigation:** Validate every request against stored token metadata, apply endpoint-level scope checks, use per-token and per-company rate limits, monitor unusual token usage patterns, and support quick revocation.

**Related docs:** [Request flow diagrams](architecture/request-flow.md), [external API access model](architecture/external-api-access-model.md), [permission model](scopes/permission-model.md), [external API rate limiting](rate-limiting/external-api-rate-limiting.md), [API key lifecycle](token-lifecycle/api-key-lifecycle.md).

### Expired or Revoked Token Reuse

**Threat:** An expired, revoked, deactivated, unknown, or incorrectly rotated key continues to be used by an integration or actor.

**Impact:** If lifecycle status is not checked at request time, stale credentials can remain usable. Even when denied, repeated attempts can create operational noise and investigation needs.

**Design-level mitigation:** Check token lifecycle status on every request, fail closed for non-active keys, return a stable unauthorized response, and audit denied credential decisions with non-secret token identifiers where available.

**Related docs:** [Invalid API key request flow](architecture/request-flow.md#invalid-expired-revoked-or-unknown-api-key), [API key lifecycle](token-lifecycle/api-key-lifecycle.md), [external API audit log](auditability/external-api-audit-log.md), [OpenAPI sample](../examples/openapi/external-api.sample.yaml).

### Arbitrary Company Access Attempts

**Threat:** A caller submits another company's identifier in a header, query parameter, request body, or resource reference to expand access beyond the token owner.

**Impact:** If caller-provided company scope is trusted, the external API can expose or mutate resources outside the token owner's company boundary.

**Design-level mitigation:** Resolve effective `CompanyId` only from token owner metadata, ignore or reject caller-provided company scope for authorization, check resource ownership inside the resolved company context, and audit company-scope violations.

**Related docs:** [External API access model](architecture/external-api-access-model.md), [request flow diagrams](architecture/request-flow.md), [permission model](scopes/permission-model.md), [external API audit log](auditability/external-api-audit-log.md).

### Excessive Request Volume or Abuse

**Threat:** A client sends aggressive polling, retry loops, invalid-token floods, high write volume, or traffic patterns that exceed expected external API usage.

**Impact:** Excessive volume can degrade external API reliability, increase downstream product load, hide credential misuse, or create avoidable operational response work.

**Design-level mitigation:** Apply layered limits by token, company, endpoint, and operation type; use stricter controls for sensitive or state-changing operations; return stable `429` responses with retry guidance; and audit rate-limited decisions.

**Related docs:** [Rate limit denial request flow](architecture/request-flow.md#rate-limit-or-abuse-detection-denial), [external API rate limiting](rate-limiting/external-api-rate-limiting.md), [external API audit log](auditability/external-api-audit-log.md), [OpenAPI sample](../examples/openapi/external-api.sample.yaml).

### Insufficient Auditability

**Threat:** Allowed, denied, lifecycle, scope-denied, rate-limited, or company-boundary decisions are missing from audit records or lack correlation IDs.

**Impact:** Operators cannot explain what happened, which token was involved, which company scope was resolved, or why a request was allowed or denied.

**Design-level mitigation:** Record structured audit events with timestamp, correlation ID, token identifier or key prefix, resolved `CompanyId`, endpoint, required scope, decision result, error classification, and response status. Keep raw tokens and sensitive payloads out of audit records.

**Related docs:** [External API audit log](auditability/external-api-audit-log.md), [request flow diagrams](architecture/request-flow.md), [API key lifecycle](token-lifecycle/api-key-lifecycle.md).

### Over-Broad Scopes

**Threat:** An API key receives broader scopes than the integration needs, such as write access for a read-only integration or wildcard-style permissions.

**Impact:** If the token is misused, the blast radius is larger and newly added endpoints may become reachable without enough review.

**Design-level mitigation:** Use explicit endpoint-level scopes, separate read and write permissions, grant least privilege, deny missing scopes by default, and review scope grants during token creation and rotation.

**Related docs:** [Scope denial request flow](architecture/request-flow.md#scope-or-permission-denial), [permission model](scopes/permission-model.md), [API key lifecycle](token-lifecycle/api-key-lifecycle.md), [OpenAPI sample](../examples/openapi/external-api.sample.yaml).

### Internal Endpoint Exposure

**Threat:** Internal controllers, internal Swagger, operational fields, or internal-only workflows are exposed through the external API surface.

**Impact:** External clients may see unstable implementation details, privileged operations, internal identifiers, or behavior that was designed only for trusted platform callers.

**Design-level mitigation:** Keep internal APIs separate from curated external API contracts, publish only reviewed external endpoints, use external DTOs, and align external documentation with stable request, response, error, scope, and rate-limit behavior.

**Related docs:** [Internal vs external API](integration-boundaries/internal-vs-external-api.md), [OpenAPI sample](../examples/openapi/external-api.sample.yaml), [permission model](scopes/permission-model.md), [request flow diagrams](architecture/request-flow.md).

### Weak Error Responses

**Threat:** Client-facing errors reveal internal service names, stack traces, token validity details beyond the documented error class, database state, capacity details, or another company's usage.

**Impact:** Attackers or curious clients can learn implementation details, improve probing attempts, or infer sensitive operational information from responses.

**Design-level mitigation:** Use stable external error shapes, include correlation IDs, keep messages safe and generic, avoid raw secrets or internal exception details, and place detailed classification in audit records rather than client responses.

**Related docs:** [Internal vs external API](integration-boundaries/internal-vs-external-api.md), [external API audit log](auditability/external-api-audit-log.md), [external API rate limiting](rate-limiting/external-api-rate-limiting.md), [OpenAPI sample](../examples/openapi/external-api.sample.yaml).

## Control Mapping

| Control | Threats reduced | Primary grouped reference |
|---|---|---|
| Company-scoped token ownership | Arbitrary company access attempts, stolen or replayed keys | [External API access model](architecture/external-api-access-model.md) |
| Request-flow ordering | Credential denial, scope denial, rate-limit denial, audited success | [Request flow diagrams](architecture/request-flow.md) |
| Store only token hashes | API key leakage | [API key lifecycle](token-lifecycle/api-key-lifecycle.md) |
| Explicit token lifecycle status | Expired or revoked token reuse | [API key lifecycle](token-lifecycle/api-key-lifecycle.md) |
| Endpoint-level scopes | Over-broad scopes, arbitrary resource access | [Permission model](scopes/permission-model.md) |
| Layered rate limits and abuse signals | Excessive request volume or abuse, replay patterns | [External API rate limiting](rate-limiting/external-api-rate-limiting.md) |
| Structured audit events | Insufficient auditability, credential misuse investigation | [External API audit log](auditability/external-api-audit-log.md) |
| External contract isolation | Internal endpoint exposure, weak error responses | [Internal vs external API](integration-boundaries/internal-vs-external-api.md) |
| Stable external error responses | Weak error responses and probing resistance | [OpenAPI sample](../examples/openapi/external-api.sample.yaml) |

## Operational Monitoring Signals

- Spike in missing, invalid, expired, revoked, or deactivated API key decisions.
- Repeated missing-scope denials for the same token, company, or endpoint.
- Repeated attempts to provide or override company scope.
- Unusual request patterns by token, company, endpoint, source IP, or user agent.
- Frequent `429 Too Many Requests` responses or soft-limit warnings.
- Write-operation retry loops or idempotency conflicts.
- External traffic to unknown, internal-looking, or undocumented routes.
- Error responses that include internal identifiers or implementation details.
- Missing audit event anomalies for known request correlation IDs.

## Common Mistakes

- Treating API key authentication as sufficient authorization.
- Trusting request-provided `CompanyId`.
- Allowing one external token to span multiple company owners.
- Granting broad scopes before endpoint-level review.
- Exposing internal Swagger or internal controllers as external documentation.
- Returning internal exception details to external clients.
- Logging raw API keys, token hashes, or sensitive payloads.
- Auditing only successful requests.
- Relying only on IP-based rate limits.
- Leaving rotated tokens active without a bounded transition plan.

## Review Checklist

- [ ] Does every external request authenticate with `X-Api-Key` before company scope is resolved?
- [ ] Is effective company scope resolved from token owner metadata only?
- [ ] Are missing, invalid, expired, revoked, and deactivated keys denied before data access?
- [ ] Does every external endpoint declare and enforce a required scope?
- [ ] Are rate limits keyed by token and company, with endpoint or operation limits where useful?
- [ ] Are allowed and denied decisions audited with correlation IDs and safe metadata?
- [ ] Are external error responses stable and free of internal details?
- [ ] Are internal APIs and internal Swagger kept separate from external contracts?
- [ ] Does the illustrative OpenAPI contract match the documented authentication and error patterns?

## Implementation Notes

This model is language-agnostic and conceptual. It does not define application code, data schemas, cloud infrastructure, or operational procedures.

Teams can adapt these design-level mitigations into security design reviews, implementation checklists, or platform risk registers.
