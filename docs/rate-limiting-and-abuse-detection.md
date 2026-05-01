# Rate limiting and abuse detection

## Purpose
Rate limiting protects external API access from:
- accidental client loops,
- credential misuse,
- excessive polling,
- brute-force token attempts,
- noisy partner integrations,
- traffic spikes affecting internal product services.

Rate limiting is not a replacement for authentication, authorization, tenant isolation, or audit logging. It limits traffic volume and highlights suspicious patterns, but every request still needs normal token validation, company resolution, scope evaluation, resource boundary checks, and audit handling.

## Why rate limiting matters in fintech external APIs
External clients are outside direct runtime control. A platform can publish integration guidance, but it cannot directly control client retry behavior, deployment defects, credential storage, or polling frequency.

Payment and status endpoints can be repeatedly polled, especially when client systems wait for asynchronous updates. Without clear limits, a small integration defect can create sustained pressure on internal services.

Leaked credentials can create high-volume traffic. Token-level and company-level limits reduce the blast radius while token revocation and investigation proceed.

One company's integration must not degrade other companies' access. Fair usage requires limits that are tied to resolved token and company ownership rather than only shared infrastructure capacity.

Excessive failed authorization attempts can indicate abuse or integration defects. Repeated missing-scope decisions, company/resource mismatch attempts, or invalid token attempts should be visible to operational teams.

Operational teams need clear throttling signals. Throttled requests should be distinguishable from authentication and authorization failures, and meaningful outcomes should be available in audit events.

## Design goals
- **Protect platform availability**: prevent external traffic from overwhelming the edge layer or downstream product services.
- **Isolate traffic by token and company**: attribute usage to the token identity and resolved company owner.
- **Preserve fair usage across companies**: prevent one company from consuming excessive shared capacity.
- **Reduce blast radius of leaked tokens**: limit how much traffic a single credential can create before revocation or containment.
- **Make abuse patterns observable**: expose repeated invalid, denied, or throttled requests for investigation.
- **Keep limits explainable to external clients**: return stable throttling responses and retry guidance.
- **Integrate rate-limit outcomes with audit logging**: record meaningful throttle decisions with token, company, endpoint, and reason.
- **Avoid relying on a single global limit**: use layered dimensions so fairness and safety are enforced close to the source of traffic.

## Non-goals
- Replacing authentication or authorization.
- Replacing fraud detection.
- Defining commercial pricing tiers.
- Implementing a complete traffic-shaping system.
- Providing production-ready code.
- Defining legal retention/compliance policy.

## Core concepts
- **Rate limit**: a rule that restricts how many requests may be accepted over a defined period.
- **Quota**: an allowed amount of usage for a token, company, endpoint, or operation class.
- **Burst limit**: a short-window cap that absorbs normal spikes while stopping sudden request floods.
- **Sustained limit**: a longer-window cap that controls ongoing request volume.
- **Token-level limit**: a limit applied to one API key identity.
- **Company-level limit**: a limit applied to the company resolved from token ownership.
- **Endpoint-level limit**: a limit applied to a specific external endpoint or route pattern.
- **Write-operation limit**: a stricter limit for operations that create or modify resources.
- **Abuse signal**: an observable pattern that may indicate misuse, leakage, attack, or integration defects.
- **Throttling response**: a stable client-facing response that indicates a request was rejected because a limit was exceeded.

## Recommended layered rate limit dimensions
### 1. Per-token limits
Per-token limits protect against leaked or misconfigured individual credentials. They ensure one API key cannot create unlimited traffic even if the owning company has broader expected usage.

### 2. Per-company limits
Per-company limits prevent one company from consuming excessive platform capacity across multiple tokens. This preserves fair usage and supports tenant-level operational controls.

### 3. Per-endpoint limits
Per-endpoint limits protect sensitive or expensive endpoints. Payment creation, payment status lookup, account listing, and webhook management may have different operational profiles.

### 4. Per-operation-type limits
Per-operation-type limits separate read, write, webhook, and payment operations. Write operations should usually be more tightly controlled than read operations because they change state or trigger downstream workflows.

### 5. Authentication failure limits
Authentication failure limits detect repeated invalid token attempts. These limits help identify brute-force attempts, misconfigured clients, or stale credentials.

### 6. Authorization deny-rate limits
Authorization deny-rate limits detect repeated scope or company-boundary violations. These patterns may indicate a client using the wrong token, an attempted cross-tenant access pattern, or an integration defect.

### 7. Global platform safety limits
Global platform safety limits are last-resort protection, not the primary fairness mechanism. They protect shared infrastructure during broad traffic spikes but should not be the only control.

## Example endpoint limit matrix
| Endpoint | Method | Primary limit dimension | Suggested behavior | Notes |
|---|---|---|---|---|
| /external/v1/banks | GET | Company and endpoint | Allow moderate read access with company attribution | Usually cacheable or low sensitivity, but still company-visible |
| /external/v1/accounts | GET | Token, company, and endpoint | Limit sustained reads per token and company | Protects account list access and downstream account queries |
| /external/v1/payments | POST | Token, company, endpoint, and write operation | Apply stricter write limits and audit throttle outcomes | Payment creation changes state and may trigger downstream workflows |
| /external/v1/payments/{paymentId} | GET | Token, company, endpoint, and resource | Limit repeated lookups for the same payment | Payment must remain within resolved company boundary |
| /external/v1/payment-status/{paymentId} | GET | Token, company, endpoint, and resource | Limit excessive polling and guide clients toward backoff | Status endpoints are common polling targets |
| /external/v1/notifications | GET | Company and endpoint | Limit sustained reads while preserving expected polling | Notifications should remain company-scoped |
| /external/v1/webhooks | POST | Token, company, endpoint, and webhook operation | Apply strict limits to registration churn | Unusual registration patterns can indicate defects or misuse |
| /external/v1/webhooks/{webhookId} | DELETE | Token, company, endpoint, and webhook operation | Apply strict limits and audit meaningful outcomes | Deletes must be limited to company-owned webhooks |

## Throttling behavior
When a request exceeds a limit, return a clear `429 Too Many Requests` response. Include retry guidance where appropriate, such as a retry interval or general backoff instruction.

Throttling responses should avoid leaking internal capacity details. They should not expose exact infrastructure thresholds, internal queue depth, or sensitive policy configuration.

Error semantics should be consistent. A throttled request should be distinguishable from authentication failure, authorization denial, and resource-not-found responses.

Meaningful throttling decisions should write audit events, especially when a token, company, endpoint, or sensitive operation crosses an expected threshold.

## Abuse detection signals
- Repeated invalid API keys.
- Repeated revoked token usage.
- Repeated expired token usage.
- Repeated missing-scope decisions.
- Repeated company/resource mismatch attempts.
- Excessive polling of payment status endpoints.
- Unusual webhook registration churn.
- Traffic spikes from one token or company.
- Requests outside expected integration pattern.

## Relationship with company-scoped API keys
Token identity and owning company are key rate-limit dimensions. Usage should be attributed to the token identity and the company resolved from token ownership.

Caller-provided `companyId` must not influence rate-limit ownership. If a token belongs to `CompanyId = 12`, rate usage should be attributed to `CompanyId = 12` regardless of any company value supplied in the request.

Token revocation should stop access regardless of remaining quota. A revoked token must be denied before any remaining allowance is treated as permission to proceed.

## Relationship with scope and permission model
Rate limiting does not grant access. It only decides whether request volume is acceptable.

Scope evaluation still applies after token validation. A request that is under its rate limit can still be denied for missing scope, expired token, revoked token, or company/resource mismatch.

Authorization failures can contribute to abuse signals. Repeated missing-scope or company-boundary denials may indicate misuse or a broken integration.

Write operations should have stricter limits than read operations. Sensitive endpoints should have explicit rate-limit expectations documented as part of their external contract.

## Relationship with audit log model
Rate-limit exceeded decisions should produce audit events. Repeated deny and throttle events should be queryable for investigation and operational response.

Audit events should include `tokenId`, `companyId`, `endpoint`, `decision`, `reason`, `requestId`, and `correlationId` where available.

Raw API keys and sensitive payloads should not be logged as part of rate-limit or abuse events.

## Client-facing error model
A conceptual `429 Too Many Requests` response should include:
| Field | Purpose |
|---|---|
| `error` | Stable error code, such as `rate_limit_exceeded`. |
| `message` | Human-readable summary that the request was throttled. |
| `retryAfterSeconds` | Optional retry guidance when the platform can provide it safely. |
| `requestId` | Support reference for the throttled request. |

The response shape should be stable and documented. Detailed internal policy configuration should not be exposed. Clients should implement backoff and avoid tight retry loops.

## Operational response model
- Observe the pattern and confirm whether it is expected integration traffic.
- Alert when repeated denied or throttled requests cross operational thresholds.
- Contact the integration owner when behavior looks accidental or misconfigured.
- Rotate or revoke a token when leakage or misuse is suspected.
- Reduce endpoint access by removing scopes when the token no longer needs them.
- Temporarily block an abusive token when immediate containment is needed.
- Review scope assignment to ensure the integration has only the access it needs.
- Tune endpoint-level limits when legitimate traffic patterns change.

## Common mistakes
- Using only a global rate limit.
- Not separating token-level and company-level limits.
- Applying the same limit to reads and writes.
- Not limiting authentication failures.
- Not logging rate-limit decisions.
- Exposing internal capacity details in error responses.
- Allowing repeated denied requests without alerting.
- Trusting caller-provided `companyId` for rate-limit ownership.
- Silently dropping throttled requests.

## Implementation notes
This model is language-agnostic and conceptual. It does not define application code.

Implementation may use API gateway policies, edge middleware, distributed counters, token bucket or leaky bucket algorithms, or dedicated traffic-control services depending on platform needs. The model does not prescribe a specific vendor or framework.
