# Security boundaries

## Internal API vs external API
Internal APIs are designed for trusted service-to-service interaction inside controlled network and identity boundaries. External APIs are designed for untrusted client environments and therefore require stricter contract, authentication, authorization, and observability controls.

Treating these as equivalent leads to control gaps and unstable integrations.

## External endpoints need explicit contracts
External consumers depend on predictable interface behavior. Each external endpoint should have explicit versioning, input/output schemas, error semantics, and authorization requirements. This reduces accidental data exposure and integration fragility.

## Keep internal Swagger local/dev-only
Automatically produced internal Swagger/OpenAPI surfaces can expose operational endpoints, experimental fields, and privileged actions not suitable for external use. Internal API docs should remain local/dev-only or restricted to internal trusted environments.

## Separate external API documentation
External API documentation should be curated and published separately from internal service docs. Separation reinforces boundary ownership, reduces accidental leakage, and allows stricter review workflows for externally visible contracts.

## External contract examples
External API contracts should be documented separately from internal APIs. Each external contract should explicitly include required scope, company/resource boundary, error model, audit expectation, and rate-limit expectation.

## Boundary between product modules and platform API
Product/domain modules should execute business logic under a resolved company scope provided by the platform boundary. They should not independently infer external caller identity. This keeps authentication and authorization policies centralized and consistent.

## Data minimization principles
External responses should include only data required for the documented use case.

Practical minimization rules:
- avoid exposing internal identifiers unless contractually needed,
- exclude debug and operational metadata from external payloads,
- apply field-level filtering where product data is broader than external entitlement,
- retain only necessary audit attributes for compliance and incident response.
