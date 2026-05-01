# ADR-0004: Use explicit token lifecycle and revocation model

- **Status**: Accepted
- **Date**: 2026-05-01

## Context

Company-scoped external API keys grant machine-to-machine access to fintech resources. Without an explicit lifecycle model, credentials can remain active for too long, become difficult to rotate, or fail to stop access reliably during incidents.

The platform needs token ownership, state, expiry, rotation, suspension, revocation, and last-used metadata to be understandable and enforceable at request time.

## Decision

Use an explicit lifecycle model for external API keys covering issuance, active state, expiry, rotation, suspension, revocation, last-used tracking, and auditable lifecycle transitions.

Token lifecycle state will be evaluated separately from company ownership, scope authorization, rate limiting, and audit logging. Raw token values will not be stored, and lifecycle changes will be recorded with actor, timestamp, and reason where appropriate.

## Consequences

Positive:

- Improves operational control.
- Reduces stale credential risk.
- Supports incident response.
- Makes token ownership and state explainable.
- Helps teams retire old integrations and review broad scopes.

Trade-offs:

- Requires lifecycle metadata management.
- Requires clear client communication for expiry and revocation.
- Requires operational ownership for rotation and incident handling.
- Requires consistent request-time enforcement rather than relying only on cleanup processes.

## Alternatives considered

1. **Boolean IsActive only**
   - Rejected: a single flag does not explain expiry, suspension, rotation, or revocation reason clearly enough.

2. **Never-expiring tokens**
   - Rejected: long-lived unmanaged credentials increase blast radius and make stale access harder to remove.

3. **Manual spreadsheet-based token tracking**
   - Rejected: manual tracking is error-prone, hard to enforce at request time, and weak for incident response.

4. **Rotating by overwriting existing token records**
   - Rejected: overwriting loses lineage and makes migration, investigation, and rollback decisions harder to explain.

5. **Revocation through delayed batch cleanup**
   - Rejected: incident response requires future authorization to stop immediately from the platform perspective.
