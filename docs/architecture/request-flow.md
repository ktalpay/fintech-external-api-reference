# Request Flow Diagrams

This document summarizes request handling for the illustrative external API contract. It is a documentation-first technical artifact, not a runtime implementation guide.

The diagrams show where authentication, company resolution, scope authorization, rate limiting, product or platform data access, audit logging, and client responses fit together. They intentionally stay concise and link to deeper design notes instead of repeating every operational detail.

## Related Documents

- [External API access model](external-api-access-model.md)
- [API key lifecycle](../token-lifecycle/api-key-lifecycle.md)
- [Permission model](../scopes/permission-model.md)
- [External API audit log](../auditability/external-api-audit-log.md)
- [External API rate limiting](../rate-limiting/external-api-rate-limiting.md)
- [Internal vs external API](../integration-boundaries/internal-vs-external-api.md)
- [Minimal illustrative OpenAPI contract](../../examples/openapi/external-api.sample.yaml)

## Ownership Rule

The core ownership rule is:

1. `X-Api-Key` resolves to the token owner.
2. The token owner determines the effective `CompanyId`.
3. External clients must not choose arbitrary company scope.

Client-provided identifiers can be request input, filters, or external resource references only after the platform has resolved trusted company scope from API key metadata.

## Successful External API Request

```mermaid
sequenceDiagram
    participant Client as External client
    participant API as External Integration API
    participant Store as API key/token store
    participant Company as Company resolution
    participant Scope as Scope/permission check
    participant Limits as Rate limit/abuse control
    participant Data as Product or platform data boundary
    participant Audit as Audit log

    Client->>API: Request with X-Api-Key
    API->>Store: Locate key prefix and validate token hash
    Store-->>API: Active token metadata with owner and scopes
    API->>Company: Resolve CompanyId from token owner
    Company-->>API: Trusted company scope
    API->>Scope: Check endpoint required scope
    Scope-->>API: Scope allowed
    API->>Limits: Apply token, company, and endpoint policy
    Limits-->>API: Request allowed
    API->>Data: Execute operation inside resolved company scope
    Data-->>API: External response model
    API->>Audit: Record allowed decision and correlation ID
    API-->>Client: 2xx response
```

This flow demonstrates the normal request path. The external client authenticates with `X-Api-Key`, but company authority comes from stored token metadata rather than request fields. Scope checks and rate-limit policy run before product or platform data is accessed, and the final decision is recorded without logging raw API key material.

## Invalid, Expired, Revoked, or Unknown API Key

```mermaid
sequenceDiagram
    participant Client as External client
    participant API as External Integration API
    participant Store as API key/token store
    participant Audit as Audit log

    Client->>API: Request with missing, unknown, expired, or revoked X-Api-Key
    API->>Store: Resolve presented key
    Store-->>API: No valid active token
    API->>Audit: Record denied credential decision
    API-->>Client: 401 Unauthorized with stable error body
```

This flow shows that credential failures stop before company resolution, scope checks, rate-limit authorization, or data access. Audit records should use a non-secret token identifier or key prefix when available and classify the denial as missing, invalid, expired, revoked, or deactivated without exposing the raw token.

## Scope or Permission Denial

```mermaid
sequenceDiagram
    participant Client as External client
    participant API as External Integration API
    participant Store as API key/token store
    participant Company as Company resolution
    participant Scope as Scope/permission check
    participant Audit as Audit log

    Client->>API: Request with valid X-Api-Key
    API->>Store: Validate token and load metadata
    Store-->>API: Active token metadata with scopes
    API->>Company: Resolve CompanyId from token owner
    Company-->>API: Trusted company scope
    API->>Scope: Compare token scopes with endpoint requirement
    Scope-->>API: Required scope missing or resource outside company scope
    API->>Audit: Record denied authorization decision
    API-->>Client: 403 Forbidden with stable error body
```

This flow demonstrates deny-by-default authorization. A valid token is not enough to call every endpoint. The token must include the required endpoint-level scope, and any resource access must remain inside the resolved company scope.

## Rate Limit or Abuse Detection Denial

```mermaid
sequenceDiagram
    participant Client as External client
    participant API as External Integration API
    participant Store as API key/token store
    participant Company as Company resolution
    participant Scope as Scope/permission check
    participant Limits as Rate limit/abuse control
    participant Audit as Audit log

    Client->>API: Request with valid X-Api-Key
    API->>Store: Validate token and load metadata
    Store-->>API: Active token metadata
    API->>Company: Resolve CompanyId from token owner
    Company-->>API: Trusted company scope
    API->>Scope: Check endpoint required scope
    Scope-->>API: Scope allowed
    API->>Limits: Evaluate token, company, endpoint, and abuse signals
    Limits-->>API: Hard limit exceeded or abuse rule matched
    API->>Audit: Record rate_limited or denied decision
    API-->>Client: 429 Too Many Requests or safe denial response
```

This flow shows rate limiting after authentication and authorization context is known. Per-token, per-company, endpoint-level, and abuse-detection policies can use trusted token and company metadata, while the client receives a stable response such as `429 Too Many Requests` when a hard limit is exceeded.
