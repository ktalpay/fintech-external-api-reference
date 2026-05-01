# ADR-0003: Use layered rate limiting for external API access

- **Status**: Accepted
- **Date**: 2026-05-01

## Context

External API traffic can originate from client systems that are outside direct runtime control. A client defect, credential leak, aggressive polling loop, or abusive request pattern can create pressure on the API boundary and downstream product services.

A single global limit does not provide enough fairness or diagnostic value for company-scoped external access. The platform needs limits that can attribute traffic to token identity, resolved company ownership, endpoint, operation type, authentication failures, and authorization deny patterns.

## Decision

Use layered rate limiting for external API access based on token identity, resolved company ownership, endpoint, operation type, authentication failures, authorization deny patterns, and platform-level safety limits.

Rate-limit outcomes should be visible to operational teams and connected to audit events where meaningful. Rate limiting does not replace authentication, authorization, company/resource boundary checks, token revocation, or audit logging.

## Consequences

Positive:

- Improves platform resilience.
- Reduces blast radius of leaked or misconfigured tokens.
- Supports fair usage across companies.
- Makes abuse patterns more observable.
- Helps distinguish throttling from authentication and authorization failures.

Trade-offs:

- Requires rate-limit state management.
- Requires clear client-facing error semantics.
- Requires operational ownership for limit tuning and incident response.
- Requires careful attribution so caller-provided company identifiers do not affect ownership.

## Alternatives considered

1. **Global-only rate limit**
   - Rejected: protects shared infrastructure only at a coarse level and does not provide token or company fairness.

2. **Per-IP-only rate limit**
   - Rejected: source network identity is unstable for many integrations and does not map reliably to token or company ownership.

3. **Endpoint-only rate limit**
   - Rejected: protects individual routes but does not prevent one token or company from consuming excessive capacity across endpoints.

4. **No rate limiting until abuse occurs**
   - Rejected: leaves the platform exposed during the first incident and gives operations teams little early warning.

5. **Relying only on downstream product service limits**
   - Rejected: allows excessive traffic to pass through the external boundary before containment and weakens external-client feedback.
