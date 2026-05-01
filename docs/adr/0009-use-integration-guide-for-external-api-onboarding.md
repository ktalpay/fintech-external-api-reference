# ADR-0009: Use integration guide for external API onboarding

- **Status**: Accepted
- **Date**: 2026-05-01

## Context

External API consumers need more than endpoint-level contracts. They also need consistent guidance for authentication, company-scoped boundaries, request identifiers, idempotency, rate limits, error handling, webhooks, token lifecycle, and safe logging.

Without a shared integration guide, clients may implement unsafe retry loops, skip idempotency, mishandle API keys, ignore stable error codes, or rely on support conversations for critical behavior.

## Decision

Maintain an implementation-neutral integration guide for external API consumers covering authentication, company-scoped boundaries, scopes, request identifiers, idempotency, rate limits, error handling, webhooks, token lifecycle, and client logging expectations.

The guide complements endpoint-specific contracts and should be reviewed when cross-cutting integration expectations change.

## Consequences

Positive:

- Improves external client onboarding.
- Reduces unsafe integration behavior.
- Makes operational expectations explicit.
- Connects architecture controls to client behavior.
- Gives support and product teams a shared reference.

Trade-offs:

- Requires maintenance as contracts evolve.
- Requires coordination between platform, product, and support teams.
- Does not replace endpoint-specific contracts.
- Requires updates when error, webhook, or token lifecycle guidance changes.

## Alternatives considered

1. **Relying only on endpoint contract examples**
   - Rejected: endpoint examples do not fully cover cross-cutting client behavior.

2. **Relying only on support conversations**
   - Rejected: verbal or ticket-based guidance is inconsistent and hard to review.

3. **Documenting only happy-path requests**
   - Rejected: clients need guidance for retries, errors, rate limits, and token lifecycle.

4. **Relying on internal engineering docs**
   - Rejected: internal docs may expose details that are not part of the external product surface.

5. **Adding guidance only after integration issues occur**
   - Rejected: external onboarding guidance should reduce preventable issues before they happen.
