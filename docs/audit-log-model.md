# Audit log model

## Purpose
Audit logs provide traceability for external API access. They should make it possible to understand:
- who called,
- which token identity was used,
- which company context was resolved,
- which endpoint/action was requested,
- whether the request was allowed or denied,
- why the decision was made.

For company-scoped API keys, audit logs are the operational record that connects token identity, resolved company ownership, scope evaluation, resource boundary checks, and request outcome.

## Why auditability matters in fintech external APIs
External clients are outside direct operational control. A platform can document contracts and enforce policies, but it cannot directly control how every partner or customer system stores credentials, rotates keys, or handles failures.

Credential leakage must be investigable. Teams need to identify which token was used, when it was used, what company context was resolved, which endpoints were attempted, and whether access was allowed or denied.

Tenant isolation decisions must be explainable. When a request is denied because it crosses a company/resource boundary, the event trail should show the resolved company context and the reason for denial.

Sensitive operations need attributable history. Payment creation, webhook management, and other externally triggered changes should be tied to a token identity and company context.

Support and security teams need reliable event trails. Audit logs help answer operational questions without relying on incomplete debug logs or scattered service traces.

Audit logs support operational investigation, but they are not a complete compliance program. Retention, access control, review processes, and regulatory interpretation require separate ownership and validation.

## Design goals
- **Append-only event trail**: audit records are written as durable events and are not casually edited in place.
- **Consistent event schema**: common fields appear across authentication, authorization, boundary, and sensitive-operation events.
- **Company-scoped attribution**: events record the company resolved from token ownership.
- **Token-level attribution without storing raw secrets**: events reference token IDs and fingerprints, never raw API keys.
- **Explicit authorization decision recording**: events record allow/deny decisions and decision reasons.
- **Low operational ambiguity**: event names, reasons, and resource references are clear enough for support and incident response.
- **Queryability for incident response**: events can be filtered by token, company, endpoint, decision, resource, time range, and correlation ID.
- **Sensitive data minimization**: events avoid full payloads and unnecessary customer or financial details.

## Non-goals
- Storing full request/response bodies.
- Replacing product/domain event history.
- Replacing security monitoring/SIEM.
- Defining legal retention requirements.
- Storing raw API keys.
- Implementing analytics dashboards.

## Audit event categories
- Authentication success.
- Authentication failure.
- Authorization allowed.
- Authorization denied.
- Scope mismatch.
- Company/resource boundary mismatch.
- Expired token usage.
- Revoked token usage.
- Rate-limit exceeded.
- Webhook management action.
- Sensitive payment action.

## Recommended audit event fields
| Field | Purpose | Required? | Notes |
|---|---|---|---|
| `eventId` | Unique identifier for the audit event. | yes | Use a stable generated ID for deduplication and support references. |
| `occurredAt` | Timestamp when the event occurred. | yes | Use synchronized clocks and a consistent timestamp format. |
| `eventType` | Category of event being recorded. | yes | Examples include `authorization.denied`, `token.revoked_usage`, or `payment.created`. |
| `decision` | Authorization outcome. | yes | Use values such as `allow`, `deny`, or `not_applicable` where no authorization decision occurred. |
| `reason` | Explanation for the decision. | yes | Examples include `scope_missing`, `company_mismatch`, `token_expired`, `token_revoked`, or `rate_limit_exceeded`. |
| `tokenId` | Non-secret token record identifier. | no | Required when a token can be resolved. Do not use the raw API key. |
| `tokenFingerprint` | Non-secret token fingerprint for correlation. | no | Should be derived safely and must not reveal the raw key. |
| `companyId` | Company context resolved from token ownership. | no | Required after successful token resolution. Do not trust caller-provided company IDs as this value. |
| `endpoint` | External API path or route template. | yes | Prefer route templates such as `/external/v1/payments/{paymentId}` over raw URLs with sensitive values. |
| `httpMethod` | HTTP method requested. | yes | Examples: `GET`, `POST`, `DELETE`. |
| `requiredScope` | Scope required by the endpoint contract. | no | Required for endpoints protected by scope evaluation. |
| `providedScopes` | Scopes associated with the resolved token. | no | May be omitted or minimized if scope lists are sensitive or large. |
| `resourceType` | Type of resource involved. | no | Examples: `bank`, `account`, `payment`, `webhook`. |
| `resourceId` | Internal resource reference involved in the request. | no | Prefer internal IDs or references; avoid sensitive external identifiers. |
| `requestId` | Request identifier at the API boundary. | yes | Used to connect audit events to operational logs. |
| `correlationId` | Cross-service correlation identifier. | no | Useful when tracing downstream product service calls. |
| `sourceIpHash` | Minimized source network reference. | no | IP addresses may be hashed or minimized depending on policy. |
| `userAgentHash` | Minimized client user-agent reference. | no | User-agent values may be hashed or minimized depending on policy. |
| `actorType` | Type of caller or credential. | yes | Example: `external_api_key`. |
| `environment` | Runtime environment where the event occurred. | yes | Examples: `sandbox`, `production`. |
| `metadata` | Controlled extension fields for non-sensitive context. | no | Must not become a dumping ground for request payloads or sensitive data. |

Raw API keys must never be logged. Request and response bodies should not be logged by default. IP and user-agent values may be hashed or minimized depending on platform policy. Metadata must be controlled so that it does not become an accidental store for sensitive payloads.

## Example audit events
### Successful bank list access
```json
{
  "eventId": "evt_001",
  "occurredAt": "2026-05-01T12:00:00Z",
  "eventType": "authorization.allowed",
  "decision": "allow",
  "reason": "scope_matched",
  "tokenId": "tok_123",
  "tokenFingerprint": "fp_abc123",
  "companyId": "company_12",
  "endpoint": "/external/v1/banks",
  "httpMethod": "GET",
  "requiredScope": "banks:read",
  "providedScopes": ["banks:read"],
  "resourceType": "bank",
  "resourceId": null,
  "requestId": "req_001",
  "correlationId": "corr_001",
  "sourceIpHash": "iphash_001",
  "userAgentHash": "uahash_001",
  "actorType": "external_api_key",
  "environment": "sandbox",
  "metadata": {
    "resultScope": "company_visible_banks"
  }
}
```

### Denied payment creation due to missing scope
```json
{
  "eventId": "evt_002",
  "occurredAt": "2026-05-01T12:05:00Z",
  "eventType": "authorization.denied",
  "decision": "deny",
  "reason": "scope_missing",
  "tokenId": "tok_123",
  "tokenFingerprint": "fp_abc123",
  "companyId": "company_12",
  "endpoint": "/external/v1/payments",
  "httpMethod": "POST",
  "requiredScope": "payments:create",
  "providedScopes": ["payments:read"],
  "resourceType": "payment",
  "resourceId": null,
  "requestId": "req_002",
  "correlationId": "corr_002",
  "sourceIpHash": "iphash_001",
  "userAgentHash": "uahash_001",
  "actorType": "external_api_key",
  "environment": "sandbox",
  "metadata": {}
}
```

### Denied request due to companyId mismatch
```json
{
  "eventId": "evt_003",
  "occurredAt": "2026-05-01T12:10:00Z",
  "eventType": "authorization.denied",
  "decision": "deny",
  "reason": "company_mismatch",
  "tokenId": "tok_123",
  "tokenFingerprint": "fp_abc123",
  "companyId": "company_12",
  "endpoint": "/external/v1/payments",
  "httpMethod": "POST",
  "requiredScope": "payments:create",
  "providedScopes": ["payments:create"],
  "resourceType": "payment",
  "resourceId": null,
  "requestId": "req_003",
  "correlationId": "corr_003",
  "sourceIpHash": "iphash_002",
  "userAgentHash": "uahash_001",
  "actorType": "external_api_key",
  "environment": "sandbox",
  "metadata": {
    "mismatchSource": "request_payload"
  }
}
```

### Revoked token usage
```json
{
  "eventId": "evt_004",
  "occurredAt": "2026-05-01T12:15:00Z",
  "eventType": "token.revoked_usage",
  "decision": "deny",
  "reason": "token_revoked",
  "tokenId": "tok_456",
  "tokenFingerprint": "fp_def456",
  "companyId": "company_34",
  "endpoint": "/external/v1/accounts",
  "httpMethod": "GET",
  "requiredScope": "accounts:read",
  "providedScopes": ["accounts:read"],
  "resourceType": "account",
  "resourceId": null,
  "requestId": "req_004",
  "correlationId": "corr_004",
  "sourceIpHash": "iphash_003",
  "userAgentHash": "uahash_002",
  "actorType": "external_api_key",
  "environment": "production",
  "metadata": {}
}
```

### Rate limit exceeded
```json
{
  "eventId": "evt_005",
  "occurredAt": "2026-05-01T12:20:00Z",
  "eventType": "rate_limit.exceeded",
  "decision": "deny",
  "reason": "rate_limit_exceeded",
  "tokenId": "tok_123",
  "tokenFingerprint": "fp_abc123",
  "companyId": "company_12",
  "endpoint": "/external/v1/payment-status/{paymentId}",
  "httpMethod": "GET",
  "requiredScope": "payment-status:read",
  "providedScopes": ["payment-status:read"],
  "resourceType": "payment",
  "resourceId": "payment_ref_789",
  "requestId": "req_005",
  "correlationId": "corr_005",
  "sourceIpHash": "iphash_001",
  "userAgentHash": "uahash_001",
  "actorType": "external_api_key",
  "environment": "production",
  "metadata": {
    "limitKey": "token"
  }
}
```

## Relationship with company-scoped API keys
The token registry resolves `companyId` from token ownership. The audit event records that resolved `companyId` so investigations can rely on the platform-authoritative company context.

Caller-provided `companyId` values must not be trusted as the audit company context. If a request includes a conflicting company value or attempts to access a resource outside the token owner's company boundary, the event should record a company/resource mismatch deny reason.

## Relationship with scope and permission model
`requiredScope` should come from the endpoint contract. The authorization decision should record whether the request was allowed or denied.

The `reason` field should record why a decision occurred, such as missing scope, expired token, revoked token, company mismatch, or rate limit. Allowed sensitive operations should still produce audit events so that important external actions have attributable history.

## Data minimization and sensitive data handling
- Never log raw API keys.
- Avoid full payload logging.
- Avoid bank account numbers, IBANs, personal data, payment descriptions, or customer identifiers unless explicitly justified.
- Prefer references, hashes, or internal IDs where appropriate.
- Separate operational diagnostics from audit events if verbose payload details are needed.

## Query and investigation use cases
- Find all calls made by a token.
- Find all denied requests for a company.
- Find repeated scope mismatch attempts.
- Identify activity after token revocation.
- Reconstruct access around a payment operation.
- Correlate an external API request with a downstream product service request.

## Operational safeguards
- Append-only write path.
- Restricted access to audit logs.
- Immutable or tamper-evident storage where appropriate.
- Clock synchronization.
- Correlation IDs across services.
- Alerting on repeated denied requests.
- Alerting on revoked token usage.
- Documented retention policy owned by the platform/security team.

## Common mistakes
- Logging raw API keys.
- Logging full request bodies by default.
- Only logging successful requests.
- Not logging deny reasons.
- Trusting caller-provided `companyId` in the audit event.
- Mixing audit events with debug logs.
- Making audit logs too hard to query.
- Storing sensitive data in free-form metadata.

## Implementation notes
This model is language-agnostic and does not define application code.

Implementation may use structured logging, event streams, append-only tables, or dedicated audit stores depending on the platform. The important requirement is a consistent, append-only audit trail that records external API access decisions without storing raw secrets or unnecessary sensitive payload data.
