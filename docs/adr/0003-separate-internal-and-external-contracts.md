# 0003: Separate Internal and External Contracts

## Status

Accepted

## Context

Internal APIs are designed for platform services, internal tools, and operational workflows. They may expose internal identifiers, broad capabilities, unstable models, and implementation-specific errors.

External clients need a curated contract that is stable, documented, company-scoped, rate-limited, auditable, and safe to publish.

## Decision

The platform maintains a separate external API contract rather than exposing internal APIs directly. External request and response DTOs, error shapes, versioning, scopes, and OpenAPI documentation are reviewed as part of the external boundary.

Internal Swagger may exist for local or controlled development use, but public external documentation should be curated separately.

## Consequences

- External clients are less coupled to internal implementation details.
- Internal refactors can occur without automatically breaking external integrations.
- External error responses and OpenAPI examples can remain stable and safe.
- Additional mapping may be required between external DTOs and application-layer models.
- New external endpoints require explicit contract review rather than simply exposing internal controllers.

## Alternatives Considered

- Publish internal Swagger as the external contract: rejected because it can expose internal behavior and unstable models.
- Reuse internal DTOs for external clients: rejected because internal model changes can become accidental breaking changes.
- Build an entirely separate external service for every capability: deferred because this reference architecture does not require a new service boundary for the initial documentation model.
