# Threat model

## Purpose
A threat model helps review external API boundaries before implementation or exposure. For company-scoped fintech APIs, it provides a structured way to reason about tenant isolation, token misuse, over-privileged access, internal API exposure, auditability gaps, rate-limit and abuse scenarios, webhook risks, and operational response.

This is a reference threat model. It is not a complete security certification or compliance assessment.

## Scope
In scope:
- external API endpoints under `/external/v1`,
- API key authentication via `X-Api-Key`,
- company-scoped authorization,
- scope and permission evaluation,
- audit logging,
- rate limiting and abuse detection,
- token lifecycle and revocation,
- webhook registration and management,
- external API contract behavior.

Out of scope:
- full cloud infrastructure threat modelling,
- full IAM program design,
- fraud detection,
- legal/regulatory compliance mapping,
- client-side application security,
- production incident response process details.

## Assets
| Asset | Why it matters | Protection expectation |
|---|---|---|
| Raw API key | Grants machine-to-machine access if presented by a caller. | Show once at creation, never store or log. |
| Token hash/fingerprint | Enables lookup and correlation without storing the raw secret. | Store securely and prevent reversal or misuse as a bearer credential. |
| Token registry | Authoritative source for token status, company ownership, scopes, expiry, and lifecycle metadata. | Restrict write access and require request-time checks. |
| Company ownership mapping | Defines tenant context for external requests. | Resolve from token metadata only; never trust caller-provided company context. |
| External API contracts | Define stable external behavior and security expectations. | Keep separate from internal APIs and review before exposure. |
| Payment request data | Can trigger sensitive financial workflows. | Minimize fields, validate strictly, audit sensitive operations. |
| Account references | Identify company-linked financial resources. | Use stable references and enforce company-scoped lookup. |
| Notification events | Inform clients about company-scoped changes. | Filter by resolved company and avoid unnecessary sensitive details. |
| Webhook URLs | Define outbound delivery targets controlled by clients. | Validate targets and prevent unsafe callback destinations. |
| Audit events | Provide traceability for access decisions and sensitive operations. | Append-only, queryable, access-controlled, and free of raw secrets. |
| Rate-limit counters | Control fairness and abuse response. | Key by token, company, endpoint, and operation where appropriate. |
| Request/correlation IDs | Connect edge requests to downstream processing and support cases. | Propagate consistently without embedding sensitive data. |

## Trust boundaries
| Boundary | What crosses the boundary | Primary risk | Required control |
|---|---|---|---|
| External client to API gateway / edge boundary | HTTPS request, API key, headers, route, parameters, payload. | Spoofed identity, malformed input, credential misuse, request flood. | TLS, token validation, input validation, rate limiting, stable error model. |
| API gateway to token registry | Token hash/fingerprint lookup and token metadata response. | Incorrect token resolution or unauthorized metadata changes. | Secure lookup, restricted registry writes, explicit token lifecycle checks. |
| API gateway to scope/permission evaluator | Resolved token, endpoint contract, required scope, and request context. | Over-privileged access or missing deny decision. | Deny-by-default scope evaluation and explicit endpoint metadata. |
| External API layer to product/domain services | Resolved company context, validated request, resource references. | Downstream service trusting caller-provided company context. | Platform-resolved company context and company-scoped product queries. |
| Product/domain services to audit log | Access decision, resource reference, request/correlation IDs, lifecycle events. | Missing audit trail or sensitive data in audit fields. | Append-only audit events and sensitive data minimization. |
| External API layer to rate-limit store | Counter keys, request classification, token/company/endpoint usage. | Unfair throttling or bypass due to weak keys. | Layered counters by token, company, endpoint, operation, and deny patterns. |
| Platform to webhook target | Outbound callback request and event payload. | Unsafe destination, data leakage, callback abuse. | Webhook target validation, company-scoped events, retry controls, audit events. |

## Threat categories
- **Spoofing**: an actor attempts to appear as a valid external client by using a stolen, guessed, expired, or revoked API key.
- **Tampering**: a caller modifies request fields such as `companyId`, resource IDs, idempotency keys, or webhook URLs to alter authorization or processing.
- **Repudiation**: a client disputes a request or operation, and the platform lacks reliable audit events to explain what happened.
- **Information disclosure**: data from another company, internal implementation details, secrets, or sensitive payload fields are exposed through responses, logs, audit metadata, or webhooks.
- **Denial of service**: accidental loops, aggressive polling, invalid-token floods, or abusive traffic degrade external APIs or downstream product services.
- **Elevation of privilege**: a token gains more access than intended through broad scopes, shared ownership, internal endpoint exposure, or downstream trust in untrusted input.

## Threat scenarios
| ID | Threat | Category | Impact | Affected boundary | Primary controls | Residual risk | Operational signal |
|---|---|---|---|---|---|---|---|
| T01 | Stolen API key is used from an unexpected environment | Spoofing | Unauthorized external access under a valid company context. | External client to API gateway / edge boundary | Token hashing, token lifecycle, audit logs, rate limiting, revocation. | Client storage quality varies. | Unusual token/company request pattern or source fingerprint. |
| T02 | Caller submits another companyId in request payload | Tampering | Cross-tenant access attempt. | External client to API gateway / edge boundary | Company-scoped ownership, deny-by-default checks, resource filtering. | Product code can drift without review. | Company-boundary violation event. |
| T03 | Token has broader scopes than required | Elevation of privilege | Larger blast radius if token is misused. | API gateway to scope/permission evaluator | Endpoint-level scopes, least privilege review, rotation scope review. | Business pressure may lead to broad grants. | Token uses endpoints outside expected pattern. |
| T04 | Expired token continues to be used | Spoofing | Old integration or leaked token keeps attempting access. | API gateway to token registry | Request-time expiry check and audit events. | Clients may not rotate on schedule. | Expired token usage events. |
| T05 | Revoked token continues to be used | Spoofing | Potential leakage or stale deployment after removal. | API gateway to token registry | Immediate revocation and alerting. | Caller may continue retrying. | Revoked token usage events. |
| T06 | Internal API endpoint is accidentally exposed externally | Information disclosure | Internal fields or privileged actions become reachable. | External client to API gateway / edge boundary | Separate internal/external surfaces and explicit contracts. | Routing errors remain possible. | External traffic to unknown or internal route pattern. |
| T07 | Payment creation endpoint is repeatedly called due to retry loop | Denial of service | Duplicate attempts and downstream pressure. | External API layer to product/domain services | Idempotency, write-operation limits, audit events. | Client retry behavior can still be defective. | Repeated idempotency keys or payment create throttles. |
| T08 | Payment status endpoint is aggressively polled | Denial of service | Excessive read load and noisy client behavior. | External API layer to rate-limit store | Endpoint and resource-level rate limits, backoff guidance. | Legitimate high-volume periods need tuning. | Repeated rate-limit exceeded events for status endpoint. |
| T09 | Webhook registration points to unsafe target | Information disclosure | Events may be sent to private or unintended destinations. | Platform to webhook target | Webhook URL validation and explicit policy. | URL ownership and network policy need ongoing review. | Validation failures or unusual webhook registration churn. |
| T10 | Audit event is missing for denied request | Repudiation | Investigation cannot explain blocked or attempted access. | Product/domain services to audit log | Append-only audit model and deny event expectations. | Logging pipeline outages can occur. | Missing audit event anomaly for request ID. |
| T11 | Audit metadata stores sensitive payload data | Information disclosure | Audit store becomes a sensitive payload repository. | Product/domain services to audit log | Sensitive data minimization and controlled metadata. | Review discipline needed as fields evolve. | Audit schema review finding or sensitive-field detection. |
| T12 | Rate-limit counters are keyed only by IP address | Denial of service | One company can consume capacity or avoid fair attribution. | External API layer to rate-limit store | Layered limits by token, company, endpoint, and operation. | Shared networks can complicate analysis. | High usage not attributable to token/company. |
| T13 | Resource lookup is performed before company boundary filtering | Information disclosure | Data from another tenant may be loaded or exposed. | External API layer to product/domain services | Resource-level company filtering and product service checks. | Query implementation mistakes remain possible. | Boundary mismatch or resource access anomaly. |
| T14 | Error response leaks internal implementation details | Information disclosure | Clients learn internal service names, exceptions, or data shape. | External client to API gateway / edge boundary | Stable client-facing error model. | Debug behavior can be misconfigured. | Error body contains internal identifiers. |
| T15 | One external token is shared across multiple companies | Elevation of privilege | Ownership and audit attribution become ambiguous. | API gateway to token registry | One-token-one-company ownership model. | Operational mistakes during issuance. | Token used for resources linked to multiple companies. |
| T16 | Downstream service trusts caller-provided companyId | Tampering | Product service can bypass platform-resolved tenant context. | External API layer to product/domain services | Resolved company context and downstream company-scoped queries. | Service boundaries need periodic review. | Requests with conflicting company fields. |
| T17 | Idempotency key is reused with conflicting payment payload | Tampering | Duplicate or inconsistent payment behavior. | External API layer to product/domain services | Idempotency scoped to token/company/endpoint and conflict response. | Client bugs can persist. | Idempotency conflict events. |
| T18 | Token rotation leaves old token active indefinitely | Elevation of privilege | Stale credential remains usable after migration. | API gateway to token registry | Explicit lifecycle, bounded rotation window, last-used tracking. | Owners may delay migration. | Old token still used after expected retirement. |

## Control mapping
| Control | Threats reduced | Related document |
|---|---|---|
| Company-scoped API key ownership | T02, T13, T15, T16 | `docs/company-scoped-api-key-model.md` |
| Token hashing/fingerprinting | T01 | `docs/company-scoped-api-key-model.md` |
| Explicit token lifecycle state | T04, T05, T18 | `docs/token-lifecycle-and-revocation.md` |
| Request-time expiry check | T04 | `docs/token-lifecycle-and-revocation.md` |
| Immediate revocation | T05, T18 | `docs/token-lifecycle-and-revocation.md` |
| Endpoint-level scopes | T03, T06 | `docs/scope-and-permission-model.md` |
| Deny-by-default authorization | T02, T03, T06, T16 | `docs/scope-and-permission-model.md` |
| Resource-level company filtering | T02, T13, T16 | `docs/scope-and-permission-model.md` |
| Explicit external API contracts | T06, T14 | `docs/external-api-contract-examples.md` |
| Append-only audit logs | T01, T05, T10, T11 | `docs/audit-log-model.md` |
| Sensitive data minimization | T11, T14 | `docs/audit-log-model.md` |
| Layered rate limiting | T01, T07, T08, T12 | `docs/rate-limiting-and-abuse-detection.md` |
| Idempotency for write operations | T07, T17 | `docs/external-api-contract-examples.md` |
| Webhook target validation | T09 | `docs/external-api-contract-examples.md` |
| Stable client-facing error model | T14 | `docs/external-api-contract-examples.md` |
| Correlation/request IDs | T10, T14 | `docs/audit-log-model.md` |
| Operational alerting | T01, T04, T05, T08, T09, T18 | `docs/rate-limiting-and-abuse-detection.md` |

## Residual risk
Some risk remains even after these controls are applied:
- client-side credential storage quality varies,
- partner retry behavior can still be defective,
- audit logs need operational review to be useful,
- rate limits require tuning,
- webhook URL validation policy needs ownership,
- insider or operational mistakes remain possible,
- contract documentation can drift unless maintained.

## Operational monitoring signals
- Spike in authentication failures.
- Repeated missing-scope denials.
- Repeated company-boundary violations.
- Revoked token usage.
- Expired token usage.
- Repeated rate-limit exceeded events.
- Webhook registration churn.
- Payment creation idempotency conflicts.
- Unusual request patterns by token/company.
- Missing audit event anomalies.

## Secure design principles used
- Least privilege.
- Deny by default.
- Defense in depth.
- Fail closed.
- Separation of internal and external surfaces.
- Tenant isolation by platform-resolved context.
- Auditability by design.
- Sensitive data minimization.
- Explicit lifecycle management.

## Common mistakes
- Treating API key authentication as sufficient authorization.
- Trusting request `companyId`.
- Exposing internal Swagger externally.
- Not modelling denied requests.
- Not threat-modelling webhook management.
- Not auditing token lifecycle events.
- Relying only on IP-based controls.
- Omitting idempotency for payment creation.
- Allowing broad scopes without review.
- Logging sensitive payloads in audit metadata.

## Implementation notes
This model is language-agnostic and conceptual. It does not define application code.

Teams may adapt this threat model into STRIDE reviews, security design reviews, architecture decision records, or platform risk registers.
