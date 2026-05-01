# Review checklist

## Purpose
This checklist helps review whether an external fintech API design is ready for exposure or architecture/security review.

It is implementation-neutral and documentation-first. It is not a production certification and not a legal/compliance checklist.

It is intended to complement the detailed architecture documents.

## How to use this checklist
Use this checklist:
- before exposing a new external endpoint,
- during architecture review,
- during security review,
- during integration onboarding review,
- when changing token, scope, webhook, error, or rate-limit behavior.

Use each item as pass/fail/needs-follow-up.

Status legend:
- **Pass**
- **Needs follow-up**
- **Not applicable**

## Review summary template
- **Review date**:
- **Reviewed API / endpoint / capability**:
- **Reviewer(s)**:
- **Status**:
- **Key risks**:
- **Required follow-ups**:
- **Decision**:

## 1. External boundary and contract review
- [ ] External endpoint is separated from internal API surface.
- [ ] Endpoint is documented as an explicit external contract.
- [ ] Method, path, request shape, response shape, and error behavior are documented.
- [ ] Internal Swagger/OpenAPI is not treated as the external contract.
- [ ] Breaking changes have versioning or migration handling.
- [ ] External contract does not expose internal service names, internal enums, stack traces, or operational endpoints.

## 2. Authentication and token ownership review
- [ ] API key is passed using the expected header.
- [ ] API key is never accepted through query string.
- [ ] Raw API key is never stored.
- [ ] Raw API key is never logged.
- [ ] Token fingerprint/hash is used for lookup or correlation.
- [ ] Token belongs to exactly one company.
- [ ] Token ownership cannot be changed to another company after issuance.
- [ ] Sandbox and production credentials are separated.

## 3. Company boundary review
- [ ] Company context is resolved from token metadata.
- [ ] Caller-provided `companyId` is treated as untrusted.
- [ ] Resource lookup is scoped to the resolved company.
- [ ] Downstream services receive platform-resolved company context.
- [ ] Product/domain services do not infer company context from external input.
- [ ] Resources outside the resolved company boundary are not disclosed.
- [ ] Invisible resources return safe behavior such as `resource_not_found` where appropriate.

## 4. Scope and permission review
- [ ] Endpoint has an explicit required scope.
- [ ] Scope naming follows the documented `resource:action` convention.
- [ ] Token must contain the required scope.
- [ ] Missing scope returns stable `insufficient_scope` behavior.
- [ ] Write operations are separated from read operations.
- [ ] Broad wildcard scopes are avoided or explicitly justified.
- [ ] Scope changes are operationally reviewed.

## 5. Token lifecycle review
- [ ] Token has explicit lifecycle state.
- [ ] Token has explicit expiry.
- [ ] Expiry is enforced at request time.
- [ ] Revoked tokens fail authorization immediately from platform perspective.
- [ ] Suspended tokens are denied.
- [ ] Rotation flow is defined.
- [ ] Last-used metadata is tracked according to policy.
- [ ] Lifecycle changes are auditable.
- [ ] Revocation reason is recorded.

## 6. Auditability review
- [ ] Authentication success/failure is auditable where appropriate.
- [ ] Authorization allowed/denied decisions are auditable.
- [ ] Company-boundary violations are auditable.
- [ ] Token lifecycle decisions are auditable.
- [ ] Rate-limit exceeded decisions are auditable when meaningful.
- [ ] Sensitive operations such as payment creation are auditable.
- [ ] Audit events include `requestId` or `correlationId` where available.
- [ ] Audit metadata excludes raw API keys and sensitive payloads.
- [ ] Audit logs are append-only or tamper-evident where appropriate.

## 7. Rate limiting and abuse detection review
- [ ] Rate limits account for token identity.
- [ ] Rate limits account for resolved company ownership.
- [ ] Endpoint-level limits exist where needed.
- [ ] Write operations have stricter limits than reads where appropriate.
- [ ] Authentication failure limits exist.
- [ ] Authorization deny-rate signals are monitored.
- [ ] `429` responses have stable error behavior.
- [ ] Clients receive safe retry guidance where available.
- [ ] Repeated abuse signals can trigger operational response.

## 8. Error handling review
- [ ] External errors use stable error codes.
- [ ] HTTP status mapping is consistent.
- [ ] `requestId` is included where available.
- [ ] Error messages do not expose internal implementation details.
- [ ] Internal exceptions are mapped to safe external errors.
- [ ] Retry semantics are documented.
- [ ] Company-boundary failures do not reveal another company's resource existence.

## 9. Idempotency review
- [ ] Write operations define idempotency behavior.
- [ ] Payment creation requires or supports `Idempotency-Key`.
- [ ] Idempotency keys are scoped to token, company, and endpoint.
- [ ] Conflicting reused keys return stable `idempotency_conflict` behavior.
- [ ] Idempotency does not bypass authentication, scope, company boundary, or validation.

## 10. Webhook review
- [ ] Webhook registration requires explicit scope.
- [ ] Webhook belongs to the resolved token owner's company.
- [ ] Webhook URL validation is defined.
- [ ] Unsafe targets such as localhost/private networks are rejected unless explicitly controlled by policy.
- [ ] Webhook payloads are signed.
- [ ] Signing secret is not logged.
- [ ] Event payloads are minimal and company-scoped.
- [ ] Duplicate delivery is expected and documented.
- [ ] Retry behavior is bounded.
- [ ] Webhook failures are observable.
- [ ] Webhook lifecycle changes are auditable.

## 11. Integration onboarding review
- [ ] Client stores API key securely.
- [ ] Client does not log raw API key.
- [ ] Client captures `requestId`.
- [ ] Client handles stable error codes.
- [ ] Client implements safe retry/backoff behavior.
- [ ] Client uses idempotency for write operations.
- [ ] Client handles rate-limit responses.
- [ ] Client verifies webhook signatures.
- [ ] Client handles duplicate webhook delivery.
- [ ] Client has token rotation and compromised-token handling process.

## 12. Threat model alignment review
- [ ] Relevant threat scenarios are identified.
- [ ] Tenant leakage risk is addressed.
- [ ] Token leakage risk is addressed.
- [ ] Over-privileged token risk is addressed.
- [ ] Internal API exposure risk is addressed.
- [ ] Audit gap risk is addressed.
- [ ] Rate-limit and abuse scenarios are addressed.
- [ ] Webhook spoofing/data leakage risks are addressed.
- [ ] Residual risks are documented.
- [ ] Operational monitoring signals are identified.

## Review outcome
| Outcome | Meaning |
|---|---|
| Ready | No blocking issues identified. |
| Ready with follow-ups | Acceptable with tracked follow-up items. |
| Not ready | Blocking risk or missing control. |

## Common review failures
- Endpoint lacks explicit required scope.
- Caller-provided `companyId` is trusted.
- Internal API is exposed directly.
- Audit expectations are missing.
- Raw API key could be logged.
- Payment creation lacks idempotency.
- Error model leaks internal detail.
- Webhook delivery is unsigned.
- Rate limiting uses only a global or IP-based limit.
- Token lifecycle relies only on `IsActive`.
- Threat model is not updated after contract changes.

## Related documents
- `docs/index.md`
- `docs/architecture-overview.md`
- `docs/security-boundaries.md`
- `docs/company-scoped-api-key-model.md`
- `docs/scope-and-permission-model.md`
- `docs/audit-log-model.md`
- `docs/rate-limiting-and-abuse-detection.md`
- `docs/token-lifecycle-and-revocation.md`
- `docs/external-api-contract-examples.md`
- `docs/error-code-reference.md`
- `docs/webhook-delivery-model.md`
- `docs/integration-guide.md`
- `docs/threat-model.md`

## Implementation notes
This checklist is language-agnostic and conceptual. It does not define application code, framework boilerplate, SDKs, or an OpenAPI/YAML file.

Teams may adapt this checklist into architecture review templates, pull request templates, security review forms, or external API readiness gates.
