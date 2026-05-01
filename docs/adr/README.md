# Architecture Decision Records

Architecture decision records explain why key external API architecture choices were made.

## Initial Decisions

- [0001: Company-scoped API keys](0001-company-scoped-api-keys.md)
- [0002: Store only token hashes](0002-store-only-token-hashes.md)
- [0003: Separate internal and external contracts](0003-separate-internal-and-external-contracts.md)

## Existing Decision Records

This repository currently contains two ADR sequences:

- the short initial sequence above, added for the public reference architecture foundation;
- the earlier `0001-use-...` sequence, which remains valid and covers related decisions in more detail.

The sequences overlap on company-scoped API keys and external contract separation, but they do not conflict. The short initial ADRs summarize foundational decisions, while the earlier ADRs preserve the existing milestone history and deeper supporting decisions about audit logging, rate limiting, token lifecycle, threat modeling, webhook delivery, error codes, integration guidance, and review readiness.
