# Problem statement: secure external API access in fintech

Fintech platforms integrate with multiple external actors: business customers, embedded finance partners, data providers, and operations tooling. External API access is necessary for growth, but it increases attack surface and amplifies tenant-isolation risk when controls are inconsistent.

## Why external API access is risky
External APIs accept traffic from systems outside direct platform control. Client runtime posture, credential handling quality, and operational discipline vary significantly across integrators. As a result, external access models must assume:
- credentials can be leaked,
- requests can be malformed or replayed,
- integration behavior can drift over time,
- and unauthorized data access attempts will occur.

## Common failure modes

### 1) Tenant leakage
If request authorization is based on caller-supplied tenant identifiers, a compromised or buggy client can request data for another company. This is a direct breach of tenant isolation.

### 2) Over-privileged tokens
Long-lived tokens with broad scopes can access unrelated resources, increasing blast radius when compromised.

### 3) Weak auditability
If API calls are not consistently logged with token identity, company ownership, and authorization outcome, forensic analysis and control verification become unreliable.

### 4) Unclear ownership boundaries
When teams cannot clearly identify which company owns a credential and what it can access, incident response slows and remediation quality drops.

### 5) Exposing internal APIs as external APIs
Internal endpoints often include broad operational functionality, unstable contracts, and assumptions about trusted network context. Directly exposing them externally creates avoidable security and reliability risk.

### 6) Poor revocation model
If revocation depends on delayed caches, manual scripts, or unclear state propagation, invalid credentials may continue to authorize requests after they should be blocked.

## Why company-scoped access matters
Company-scoped access ties each token to a single owning company and evaluates every request against that ownership. This enforces tenant isolation at the authorization boundary and prevents cross-company data access even when request payloads attempt to reference another company.

In practical terms, company-scoped access changes the trust model:
- trust the token-to-company mapping from the platform registry,
- do not trust company identifiers supplied by external callers,
- and require all downstream service calls to carry platform-resolved company context.
