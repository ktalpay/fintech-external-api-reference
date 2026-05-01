# ADR 0001: Use company-scoped API keys for external API access

- **Status**: Accepted
- **Date**: 2026-05-01

## Context

The platform requires a secure and operationally manageable pattern for authenticating and authorizing external API requests across multiple tenant companies. Previous industry patterns often fail by trusting tenant identifiers supplied by external clients or by issuing overly broad credentials detached from clear ownership.

These patterns increase risk of cross-tenant access, weak forensic traceability, and difficult revocation during incidents.

## Decision

Adopt company-scoped API keys as the default external access model.

Decision details:

1. Each key is bound to exactly one owning company.
2. Authorization company context is resolved from token ownership metadata, not request payload.
3. Scopes/permissions are evaluated per endpoint/action before invoking product services.
4. External request handling emits audit events with token reference, company scope, decision outcome, and timestamp.
5. Token lifecycle controls include explicit expiration, revocation, and last-used tracking.

## Consequences

Positive:

- Stronger tenant isolation at the API boundary.
- Clear credential ownership and accountability.
- Better incident response due to auditable token activity.
- Reduced probability of accidental cross-company data access.

Trade-offs:

- Requires a reliable token registry and policy evaluation component.
- Introduces operational overhead for key issuance, rotation, and revocation workflows.
- May require endpoint redesign where existing contracts assume caller-supplied tenant context.

## Alternatives considered

1. **Caller-supplied companyId with shared credentials**
   - Rejected: high tenant-leakage risk and poor accountability.

2. **Static allowlists by source IP only**
   - Rejected: insufficient identity assurance and poor portability for modern integrations.

3. **Direct exposure of internal service credentials/endpoints**
   - Rejected: weak boundary control and unstable external contract.
