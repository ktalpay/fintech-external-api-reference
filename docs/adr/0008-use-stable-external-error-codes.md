# ADR-0008: Use stable external error codes

- **Status**: Accepted
- **Date**: 2026-05-01

## Context

External API clients need predictable failure semantics. If the same condition produces different error codes, unsafe details, or unclear retry behavior, integrations become harder to build and support.

Company-scoped external APIs also need error responses that do not leak internal implementation details or reveal resources outside the resolved company boundary.

## Decision

Use stable, documented external error codes with consistent HTTP status mapping, safe client-facing messages, `requestId` correlation, minimal structured details, and no internal implementation leakage.

Internal exceptions and product-domain errors should be mapped to the external error model before responses are returned to clients.

## Consequences

Positive:

- Improves client integration behavior.
- Improves support and investigation workflows.
- Reduces accidental disclosure of internal details.
- Clarifies retry behavior.
- Makes endpoint contracts easier to review.

Trade-offs:

- Requires maintenance discipline.
- Requires coordination across product/platform teams.
- Requires careful mapping from internal exceptions to external errors.
- Requires review when new endpoint contracts add new failure modes.

## Alternatives considered

1. **Raw internal exception messages**
   - Rejected: exposes implementation details and creates unstable external behavior.

2. **Generic internal_error for most failures**
   - Rejected: prevents clients from handling authentication, authorization, validation, idempotency, and throttling predictably.

3. **Undocumented error behavior**
   - Rejected: makes client integrations fragile and support investigations harder.

4. **HTTP status codes only**
   - Rejected: status codes alone do not provide enough stable machine-readable detail.

5. **Exposing internal product error enums directly**
   - Rejected: internal enums may change and may reveal internal product structure.
