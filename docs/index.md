# Documentation Index

## Reading path

Recommended reading order:
1. Problem statement
2. Architecture overview
3. Security boundaries
4. Company-scoped API key model
5. Scope and permission model
6. Audit log model
7. Rate limiting and abuse detection
8. Token lifecycle and revocation
9. External API contract examples
10. Threat model
11. Review checklist
12. Architecture decision records

## Structured Entry Points

| Section | Focus |
|---|---|
| [Architecture](architecture/index.md) | External API access model and request flow. |
| [Security](security/index.md) | Security principles and trust boundaries. |
| [Integration boundaries](integration-boundaries/index.md) | Internal API vs external API contract separation. |
| [Token lifecycle](token-lifecycle/index.md) | API key generation, storage, rotation, expiry, and revocation. |
| [Auditability](auditability/index.md) | External API audit logging strategy. |
| [Rate limiting](rate-limiting/index.md) | Per-token and per-company rate limiting strategy. |
| [Scopes](scopes/index.md) | Endpoint-level permission model. |
| [Architecture decision records](adr/README.md) | Architecture decision record index. |

## Core architecture documents

| Document | Focus | When to read |
|---|---|---|
| `docs/problem-statement.md` | External API risk framing and failure modes. | Start here to understand the problem this reference architecture addresses. |
| `docs/architecture-overview.md` | High-level actors, request flow, and control points. | Read before reviewing detailed security models. |
| `docs/architecture/external-api-access-model.md` | Core external API request model and company scope resolution. | Read when designing API key based request authorization. |
| `docs/security-boundaries.md` | Internal/external API separation and boundary guidance. | Read when deciding what should be exposed externally. |
| `docs/integration-boundaries/internal-vs-external-api.md` | Internal API vs external API contract boundary strategy. | Read when deciding how to expose or document APIs externally. |
| `docs/company-scoped-api-key-model.md` | Token ownership and company resolution. | Read when designing tenant-scoped API key authorization. |
| `docs/scope-and-permission-model.md` | Endpoint scopes, permission checks, and resource boundaries. | Read when assigning endpoint-level access. |
| `docs/scopes/permission-model.md` | Pragmatic scope naming and endpoint-level permission model. | Read when defining external scopes. |
| `docs/audit-log-model.md` | Audit event categories, fields, examples, and data minimization. | Read when designing traceability and investigation support. |
| `docs/auditability/external-api-audit-log.md` | Audit log strategy for external API requests and token events. | Read when defining audit fields and decision results. |
| `docs/rate-limiting-and-abuse-detection.md` | Layered limits and abuse signals. | Read when designing throttling and operational response. |
| `docs/rate-limiting/external-api-rate-limiting.md` | Per-token, per-company, burst, and 429 response guidance. | Read when defining external API rate-limit policy. |
| `docs/token-lifecycle-and-revocation.md` | Token issuance, expiry, rotation, suspension, revocation, and last-used tracking. | Read when designing API key operations. |
| `docs/token-lifecycle/api-key-lifecycle.md` | API key lifecycle, hashed storage, and operational failure cases. | Read when defining token lifecycle behavior. |
| `docs/external-api-contract-examples.md` | Example endpoint contracts and error conventions. | Read when documenting externally visible endpoints. |
| `examples/openapi/external-api.sample.yaml` | Minimal OpenAPI contract for generic external endpoints. | Read when reviewing example external API documentation. |
| `docs/threat-model.md` | Threat scenarios, controls, residual risks, and monitoring signals. | Read during security review or release readiness review. |
| `docs/review-checklist.md` | Implementation-neutral readiness checklist. | Use for endpoint readiness review before exposure or major contract changes. |

## Architecture decision records

| ADR | Decision | Related topic |
|---|---|---|
| `docs/adr/0001-company-scoped-api-keys.md` | Use company-scoped API keys for external API access. | Token ownership and company scope. |
| `docs/adr/0002-store-only-token-hashes.md` | Store only token hashes. | Token storage and credential exposure risk. |
| `docs/adr/0003-separate-internal-and-external-contracts.md` | Separate internal APIs from external API contracts. | Integration boundaries and contract stability. |
| `docs/adr/0001-use-company-scoped-api-keys.md` | Use company-scoped API keys for external API access. | Token ownership and tenant isolation. |
| `docs/adr/0002-use-append-only-audit-log-for-external-api-access.md` | Use append-only audit events for external API access. | Audit logging and investigation. |
| `docs/adr/0003-use-layered-rate-limiting-for-external-api-access.md` | Use layered rate limiting for external API access. | Availability, fairness, and abuse detection. |
| `docs/adr/0004-use-explicit-token-lifecycle-and-revocation-model.md` | Use explicit token lifecycle and revocation controls. | Token operations and incident response. |
| `docs/adr/0005-use-explicit-external-api-contracts.md` | Use explicit external API contracts. | External endpoint documentation. |
| `docs/adr/0006-use-threat-model-for-external-api-boundaries.md` | Maintain a threat model for external API boundaries. | Security review and residual risk. |
| `docs/adr/0010-use-review-checklist-for-external-api-readiness.md` | Use a review checklist for external API readiness. | Endpoint readiness and cross-cutting control review. |

## Suggested reviewer checklist

- [ ] Is company ownership resolved from token metadata?
- [ ] Are caller-provided company identifiers treated as untrusted?
- [ ] Does every external endpoint have an explicit required scope?
- [ ] Are denied authorization decisions auditable?
- [ ] Are raw API keys excluded from storage and logs?
- [ ] Are expired, revoked, and suspended tokens denied at request time?
- [ ] Are rate limits applied by token, company, endpoint, and operation type where appropriate?
- [ ] Are write operations idempotent where needed?
- [ ] Are internal APIs separated from external contracts?
- [ ] Are threat scenarios mapped to controls?

## Repository status

Early reference architecture documentation set.

Release notes:
- `docs/release-notes/v0.1.0.md`
- `docs/release-notes/v0.2.0.md`
