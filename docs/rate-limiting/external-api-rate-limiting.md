# External API Rate Limiting

External API rate limiting protects platform reliability, reduces abuse risk, and gives operators clear controls when integrations behave unexpectedly.

Rate limiting should be layered. A single global limit is rarely enough for company-scoped external API access.

## Per-Token and Per-Company Limits

Per-token limits control the behavior of one API key. They are useful when a specific integration is too noisy or compromised.

Per-company limits control aggregate usage across all keys owned by a company. They prevent a company from bypassing limits by creating many tokens or splitting traffic across integrations.

Both are needed:

- per-token limits isolate individual credentials,
- per-company limits protect shared tenant-level capacity,
- endpoint-level limits protect sensitive or expensive operations,
- write-operation limits reduce risk from repeated state changes.

## Soft Limit vs Hard Limit

A soft limit warns or records that traffic is approaching an operational threshold. It may trigger dashboards, alerts, or customer outreach without rejecting requests.

A hard limit rejects requests after the threshold is exceeded. Hard limits should return a stable `429 Too Many Requests` response and should be visible in audit and operational metrics.

Using both allows teams to detect problematic trends before integrations break.

## Burst Limits

Burst limits allow short spikes while still controlling sustained usage.

Example approach:

- allow a small burst for brief retries or batch startup,
- apply a sustained per-minute or per-hour quota,
- use stricter burst controls for state-changing endpoints,
- avoid making burst capacity so large that it defeats the limit.

The exact algorithm can vary by implementation. Token bucket, leaky bucket, fixed window, and sliding window approaches are all possible choices.

## Abuse Detection

Rate limiting should feed abuse detection signals, such as:

- repeated invalid API key attempts,
- high denied-request volume,
- many requests to sensitive endpoints,
- unexpected IP address changes,
- usage outside historical patterns,
- repeated retries without backoff,
- attempts to access resources outside company scope.

Abuse signals can lead to stricter limits, temporary deactivation, manual review, or token revocation.

## 429 Response Design

`429 Too Many Requests` responses should be stable and safe for external clients.

Example response shape:

```json
{
  "error": {
    "code": "rate_limit_exceeded",
    "message": "Too many requests. Retry after the indicated interval.",
    "correlation_id": "req_01HZXAMPLE000000000000000"
  }
}
```

The response should not reveal internal capacity details, database state, or other companies' usage.

## Retry-After Header

When possible, include a `Retry-After` header. The value can be a delay in seconds or an HTTP date.

Clients should treat `Retry-After` as the minimum wait before retrying. Documentation should recommend exponential backoff and jitter, especially for repeated `429` responses.

## Operational Dashboards

Operational dashboards should show:

- request volume by company scope,
- request volume by token identifier,
- rate-limited requests by endpoint,
- denied requests by error classification,
- token usage trends,
- top clients approaching soft limits,
- hard-limit events over time,
- unusual IP or user-agent patterns.

Dashboards should use token identifiers, key prefixes, and company scope metadata. They must not display raw API key tokens.

## Policy Review

Rate-limit policies should be reviewed when:

- new external endpoints are added,
- write operations are introduced,
- an integration's normal traffic pattern changes,
- abuse signals increase,
- operational dashboards show frequent throttling,
- client support reports repeated `429` responses.

Policy changes should be documented and, when possible, rolled out gradually with monitoring.
