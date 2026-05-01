# ADR-0005: Use explicit external API contracts

- **Status**: Accepted
- **Date**: 2026-05-01

## Context

External API clients need stable, reviewable contracts that are separated from internal service APIs. Internal API surfaces can include operational actions, experimental fields, internal identifiers, or behavior that should not be exposed to partner or customer systems.

Company-scoped access also requires each endpoint contract to make security boundaries explicit. A contract that omits required scope, company/resource boundary, error behavior, audit expectations, or rate-limit expectations leaves too much to implementation assumptions.

## Decision

External APIs must be documented as explicit contracts separated from internal API surfaces. Each external contract should define required scope, company/resource boundary, request/response shape, error behavior, audit expectations, rate-limit expectations, and idempotency behavior where relevant.

These contracts are part of the external product surface and should be reviewed alongside security boundaries, scope policy, audit expectations, and operational controls.

## Consequences

Positive:

- Improves external integration clarity.
- Reduces accidental exposure of internal APIs.
- Makes security boundaries reviewable.
- Improves support and incident handling.
- Helps product and platform teams agree on stable external behavior.

Trade-offs:

- Requires contract maintenance discipline.
- Requires versioning and migration ownership.
- Requires coordination between platform and product teams.
- Requires error and audit behavior to be treated as part of the endpoint contract.

## Alternatives considered

1. **Exposing internal APIs directly**
   - Rejected: internal APIs are not designed as stable external contracts and can expose unsafe behavior or fields.

2. **Relying on internal Swagger/OpenAPI only**
   - Rejected: internal API references do not necessarily describe external scopes, company boundaries, audit expectations, or rate-limit behavior.

3. **Documenting endpoints without authorization boundaries**
   - Rejected: clients and reviewers need to understand required scope and company/resource boundary for each endpoint.

4. **Treating error behavior as implementation detail**
   - Rejected: stable error semantics are necessary for client integrations, support, and incident response.

5. **Adding contracts only after clients request them**
   - Rejected: contracts should be established before exposure so security and operational behavior can be reviewed early.
