# ADR-0010: Use review checklist for external API readiness

- **Status**: Accepted
- **Date**: 2026-05-01

## Context

External API readiness depends on multiple cross-cutting controls: explicit contracts, company-scoped ownership, endpoint scopes, token lifecycle, auditability, rate limiting, error handling, idempotency, webhooks, integration onboarding, and threat model alignment.

Detailed documents describe each area, but reviewers also need a concise way to check whether a specific endpoint or capability is ready for exposure.

## Decision

Maintain an implementation-neutral review checklist for external API readiness covering external contracts, company-scoped ownership, scopes, token lifecycle, auditability, rate limiting, error handling, idempotency, webhooks, integration onboarding, and threat model alignment.

The checklist should be used before new external exposure and when changing cross-cutting behavior.

## Consequences

Positive:

- Improves review consistency.
- Makes cross-cutting controls easier to validate.
- Supports architecture/security/product alignment.
- Helps identify missing controls before exposure.
- Gives reviewers a concise readiness artifact.

Trade-offs:

- Requires maintenance as the reference architecture evolves.
- Does not replace detailed security testing.
- Does not replace endpoint-specific design review.
- Requires reviewers to interpret checklist items in context.

## Alternatives considered

1. **Relying only on detailed documents**
   - Rejected: detailed documents are useful, but reviewers need a concise readiness workflow.

2. **Relying only on reviewer experience**
   - Rejected: review quality becomes inconsistent and harder to audit.

3. **Reviewing after exposure**
   - Rejected: missing controls should be found before external release.

4. **Using a generic security checklist only**
   - Rejected: company-scoped fintech APIs need specific tenant, token, scope, webhook, and contract checks.

5. **Embedding review expectations only inside individual docs**
   - Rejected: scattered checklist items are harder to apply during readiness review.
