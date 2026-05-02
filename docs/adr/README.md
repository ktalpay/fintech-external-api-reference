# Architecture Decision Records

Architecture decision records explain why key external API architecture choices were made for this documentation-first technical artifact.

Return to the [documentation index](../index.md).

## Historical Numbering

The ADR folder contains two historical sequences:

- Short initial sequence: `0001-company-scoped-api-keys.md` through `0003-separate-internal-and-external-contracts.md`.
- Earlier explicit `use-*` sequence: `0001-use-company-scoped-api-keys.md` through `0010-use-review-checklist-for-external-api-readiness.md`.

The repeated numbers are historical and do not mean the decisions conflict. The short initial sequence summarizes foundational reference architecture decisions. The `use-*` sequence preserves earlier milestone decisions with more explicit names and supporting topics such as audit logging, rate limiting, token lifecycle, threat modeling, webhook delivery, stable errors, integration guidance, and review readiness.

Do not infer precedence from filename number alone. Read the filename and title together.

## Recommended Reading Order

1. Read the short initial sequence for the compact foundation:
   - [0001: Company-scoped API keys](0001-company-scoped-api-keys.md)
   - [0002: Store only token hashes](0002-store-only-token-hashes.md)
   - [0003: Separate internal and external contracts](0003-separate-internal-and-external-contracts.md)
2. Read the explicit `use-*` sequence for the broader decision history:
   - [0001: Use company-scoped API keys](0001-use-company-scoped-api-keys.md)
   - [0002: Use append-only audit log](0002-use-append-only-audit-log-for-external-api-access.md)
   - [0003: Use layered rate limiting](0003-use-layered-rate-limiting-for-external-api-access.md)
   - [0004: Use explicit token lifecycle and revocation model](0004-use-explicit-token-lifecycle-and-revocation-model.md)
   - [0005: Use explicit external API contracts](0005-use-explicit-external-api-contracts.md)
   - [0006: Use threat model for external API boundaries](0006-use-threat-model-for-external-api-boundaries.md)
   - [0007: Use signed webhook delivery model](0007-use-signed-webhook-delivery-model.md)
   - [0008: Use stable external error codes](0008-use-stable-external-error-codes.md)
   - [0009: Use integration guide for external API onboarding](0009-use-integration-guide-for-external-api-onboarding.md)
   - [0010: Use review checklist for external API readiness](0010-use-review-checklist-for-external-api-readiness.md)

## Short Initial Sequence

| ADR | One-line description |
|---|---|
| [0001: Company-scoped API keys](0001-company-scoped-api-keys.md) | Establishes company-scoped API keys as the external access foundation. |
| [0002: Store only token hashes](0002-store-only-token-hashes.md) | Records the decision to store verification artifacts rather than raw API keys. |
| [0003: Separate internal and external contracts](0003-separate-internal-and-external-contracts.md) | Records the decision to keep external contracts separate from internal API surfaces. |

## Explicit Use-* Sequence

| ADR | One-line description |
|---|---|
| [0001: Use company-scoped API keys](0001-use-company-scoped-api-keys.md) | Defines company-scoped API keys for external API access and tenant isolation. |
| [0002: Use append-only audit log](0002-use-append-only-audit-log-for-external-api-access.md) | Records audit logging as a traceability control for external API decisions. |
| [0003: Use layered rate limiting](0003-use-layered-rate-limiting-for-external-api-access.md) | Records layered rate limiting for token, company, endpoint, and abuse-control concerns. |
| [0004: Use explicit token lifecycle and revocation model](0004-use-explicit-token-lifecycle-and-revocation-model.md) | Records lifecycle states and revocation behavior for external API keys. |
| [0005: Use explicit external API contracts](0005-use-explicit-external-api-contracts.md) | Records the decision to publish curated external API contracts. |
| [0006: Use threat model for external API boundaries](0006-use-threat-model-for-external-api-boundaries.md) | Records the decision to maintain an illustrative threat model for external API boundaries. |
| [0007: Use signed webhook delivery model](0007-use-signed-webhook-delivery-model.md) | Records signed webhook delivery as the reference approach for webhook events. |
| [0008: Use stable external error codes](0008-use-stable-external-error-codes.md) | Records stable external error codes for predictable client-facing behavior. |
| [0009: Use integration guide for external API onboarding](0009-use-integration-guide-for-external-api-onboarding.md) | Records an integration guide as onboarding support for external API clients. |
| [0010: Use review checklist for external API readiness](0010-use-review-checklist-for-external-api-readiness.md) | Records a review checklist for endpoint readiness and cross-cutting control review. |

## Notes for Readers

- The ADRs document design decisions for a non-production reference architecture.
- The ADRs are intentionally generic and do not include real company, customer, or proprietary details.
- New documentation should prefer linking to this index rather than assuming a single uninterrupted ADR number sequence.
