# ADR-0006: Use threat model for external API boundaries

- **Status**: Accepted
- **Date**: 2026-05-01

## Context

Company-scoped external API access crosses several sensitive boundaries: untrusted clients call platform endpoints, tokens resolve tenant context, scope decisions gate operations, product services execute company-scoped work, audit logs record decisions, and webhook targets receive outbound events.

These boundaries need explicit threat/control reasoning before endpoints are exposed. Without a documented model, teams may implement controls without a shared view of the threats they reduce or the residual risks that remain.

## Decision

Maintain a threat model for company-scoped external API access that maps external API risks to controls such as company-scoped token ownership, endpoint scopes, audit logging, rate limiting, token lifecycle controls, explicit contracts, idempotency, and webhook validation.

The threat model will be reviewed as external API contracts evolve and should inform architecture reviews, endpoint design, and operational monitoring priorities.

## Consequences

Positive:

- Improves security review quality.
- Makes threat/control reasoning explicit.
- Helps prioritize external API safeguards.
- Supports onboarding and architecture review.
- Gives product and platform teams shared language for boundary risks.

Trade-offs:

- Requires maintenance as contracts evolve.
- Requires ownership for residual risk review.
- Does not replace formal security testing or compliance review.
- Requires teams to keep controls and documented contracts aligned.

## Alternatives considered

1. **Relying only on implementation review**
   - Rejected: implementation review can miss boundary-level assumptions and cross-document control gaps.

2. **Documenting controls without threats**
   - Rejected: controls are harder to prioritize or validate without the threats they reduce.

3. **Threat-modelling only after incidents**
   - Rejected: late review increases exposure and weakens prevention.

4. **Treating gateway configuration as sufficient**
   - Rejected: gateway controls do not cover product service behavior, audit expectations, token lifecycle, idempotency, or webhook risks by themselves.

5. **Relying on generic platform security guidance only**
   - Rejected: company-scoped external API access has specific tenant, token, contract, and operational risks that need dedicated review.
