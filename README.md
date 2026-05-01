# fintech-external-api-reference

Public reference architecture for secure external API access in fintech platforms.

## Project Purpose

This repository documents a secure, documentation-first approach to external API access for fintech-style platforms. It focuses on API key lifecycle management, company-scoped external access, auditability, rate limiting, token revocation, permission scopes, and integration boundaries.

The goal is to make the architecture understandable before any implementation is chosen. The examples are intentionally generic and vendor-neutral.

## Problem Addressed

External API integrations often need access to sensitive platform capabilities, such as listing available banks, reading account data, creating payment requests, or managing webhooks. If company scope, token ownership, permission checks, and audit records are unclear, external access can become difficult to operate and risky to investigate.

This repository describes a reference model where:

- external clients authenticate with platform-issued API keys,
- the platform resolves company scope from trusted token metadata,
- every request is evaluated against endpoint-level scopes,
- token lifecycle events are auditable,
- rate limits reduce operational and abuse risk,
- internal APIs remain separate from curated external API contracts.

## Core Concepts

| Concept | Description |
|---|---|
| API key | A platform-issued credential used by an external client. |
| Token hash | The stored representation of an API key token. Raw tokens are displayed once and are not stored. |
| Key prefix | A non-secret API key identifier used for lookup, support, and audit correlation. |
| Company scope | The company ownership context resolved from API key metadata. |
| External client | A system outside the platform boundary that calls the external API. |
| External API contract | A curated, stable API surface intended for external clients. |
| Internal API | A private API surface used by platform operators, internal tools, or internal services. |
| Scope | A named permission that authorizes a specific class of external API operation. |
| Audit log | An append-only record of security-relevant external API decisions and lifecycle events. |

## Repository Structure

| Path | Purpose |
|---|---|
| `docs/architecture/` | Core architecture models and request flow documentation. |
| `docs/security/` | Security principles and boundary guidance. |
| `docs/integration-boundaries/` | Internal vs external API contract guidance. |
| `docs/token-lifecycle/` | API key issuance, storage, rotation, expiry, and revocation guidance. |
| `docs/auditability/` | Audit logging guidance for external API access. |
| `docs/rate-limiting/` | Rate limiting and abuse detection guidance. |
| `docs/scopes/` | Scope and permission model guidance. |
| `docs/adr/` | Architecture decision records. |
| `examples/` | Minimal illustrative examples, including OpenAPI contracts. |

Additional documentation from earlier milestones remains available under `docs/`.

## Documentation Entry Points

- [Architecture](docs/architecture/index.md)
- [Security](docs/security/index.md)
- [Integration boundaries](docs/integration-boundaries/index.md)
- [Token lifecycle](docs/token-lifecycle/index.md)
- [Auditability](docs/auditability/index.md)
- [Rate limiting](docs/rate-limiting/index.md)
- [Scopes](docs/scopes/index.md)
- [Architecture decision records](docs/adr/README.md)
- [Examples](examples/README.md)
- [Minimal external OpenAPI sample](examples/openapi/external-api.sample.yaml)

## Non-Goals

- This is not a banking compliance framework.
- This is not a complete production implementation.
- This does not provide legal, regulatory, or audit advice.
- This is not tied to any real company or customer.
- This does not define a cloud-provider deployment model.
- This does not add backend services, database migrations, or authentication libraries.

## Status

Early reference architecture.

The repository is documentation-first and intentionally implementation-light. Documents and examples are suitable for public review, adaptation, and discussion, but they are not a substitute for production engineering, security review, or organization-specific governance.

## Disclaimer

This repository provides generic reference architecture documentation only. It does not provide production, legal, compliance, regulatory, audit, security certification, or financial advice, and it does not guarantee suitability for any regulated environment.
