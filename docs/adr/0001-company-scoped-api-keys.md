# 0001: Company-Scoped API Keys

## Status

Accepted

## Context

External API clients need a credential that authorizes access to platform capabilities. In a fintech-style platform, those capabilities must be isolated by company so that one external client cannot access another company's data or operations.

Relying on client-provided company identifiers would make authorization ambiguous and could allow accidental or malicious cross-company access attempts.

## Decision

External API keys are company-scoped. Each API key record belongs to one company, and request authorization resolves `CompanyId` from trusted API key metadata.

External clients must not provide or override `CompanyId` for authorization. Any business identifiers supplied by the client are evaluated only inside the resolved company scope.

## Consequences

- Company isolation is enforced consistently at the external API boundary.
- Audit logs can attribute requests to a token identifier and resolved company scope.
- Rate limits can be applied per token and per company.
- Support tooling can reason about API key ownership without trusting caller input.
- Key transfer between companies is not supported; a new key must be issued for a different company scope.

## Alternatives Considered

- Client-provided `CompanyId`: rejected because caller-provided tenant identity is not trusted authority.
- Global external API keys: rejected because they increase blast radius and weaken auditability.
- Per-request company selection by the client: rejected because it complicates authorization and creates cross-company access risk.
