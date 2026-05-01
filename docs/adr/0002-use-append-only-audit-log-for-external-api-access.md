# ADR-0002: Use append-only audit log for external API access

- **Status**: Accepted
- **Date**: 2026-05-01

## Context

External API access creates security and operational questions that must be answerable after the request is complete. Teams need to know which token identity was used, which company context was resolved, which endpoint was requested, whether authorization was allowed or denied, and why that decision occurred.

Debug logs, gateway logs, and product-domain history can each provide useful signals, but none of them alone establishes a consistent authorization-focused record for external access. The platform needs a dedicated audit model that connects authentication, scope evaluation, company/resource boundaries, sensitive operations, and rate-limit outcomes.

## Decision

Use an append-only audit event model for external API authentication, authorization, company/resource boundary decisions, sensitive operations, and rate-limit outcomes.

Audit events will record non-secret token references, resolved company context, endpoint/action information, required scopes, decisions, reasons, resource references where appropriate, request/correlation identifiers, and controlled metadata. Raw API keys and full request/response payloads are not part of the default audit event model.

## Consequences

Positive:

- Improves traceability for external API access.
- Supports incident response when credentials are leaked, revoked, misused, or overprivileged.
- Makes tenant boundary and scope decisions easier to explain.
- Gives support and security teams a consistent event trail for investigation.

Trade-offs:

- Creates storage and query design responsibility.
- Requires sensitive data minimization in event schema and metadata.
- Requires retention and access-control decisions owned by the appropriate platform/security teams.
- Adds operational expectations for clock synchronization, correlation IDs, and audit event availability.

## Alternatives considered

1. **Application debug logs only**
   - Rejected: debug logs are often inconsistent, noisy, short-lived, and not designed as an authorization record.

2. **Product-domain event history only**
   - Rejected: product events may capture business changes, but they may not record denied requests, token identity, required scope, or company boundary decisions.

3. **Storing full request/response payloads**
   - Rejected: this increases sensitive data exposure and makes audit storage harder to control.

4. **Relying only on gateway logs**
   - Rejected: gateway logs may capture request routing, but they may not include platform-resolved company context, endpoint scope requirements, downstream resource ownership checks, or sensitive-operation attribution.
