# Integration guide

## Purpose
This guide describes how an external client should integrate with a company-scoped fintech external API.

It is implementation-neutral and is not production code. It complements the contract examples, error code reference, webhook model, and security boundary documents.

## Integration assumptions
- Client has been issued an API key.
- Token belongs to exactly one company.
- Access is limited by scopes.
- Company context is resolved from token ownership.
- Caller-provided `companyId` is not authoritative.
- External contracts are versioned under `/external/v1`.
- Clients should handle retries, idempotency, and rate limits safely.

## High-level integration flow
1. Obtain API key through the platform-approved process.
2. Store API key securely.
3. Confirm assigned scopes.
4. Call read endpoint with `X-Api-Key`.
5. Use `requestId` for support correlation.
6. Use `Idempotency-Key` for write operations.
7. Handle errors using stable error codes.
8. Respect rate-limit responses.
9. Register webhook endpoint if asynchronous updates are needed.
10. Verify webhook signatures.
11. Rotate token before expiry.
12. Revoke unused or compromised tokens.

## Authentication
Use the `X-Api-Key` header.

Never place API key in URL query string. Never log raw API key.

Separate sandbox and production credentials. Rotate credentials according to lifecycle policy. Treat token ownership as company-scoped.

Conceptual request example:
```text
GET /external/v1/banks
X-Api-Key: <api-key>
```

## Scope and permission expectations
Each endpoint has a required scope. `insufficient_scope` means the token does not have the scope required by that endpoint.

An active token alone does not grant endpoint access. Clients should request only needed scopes, and scope changes should be reviewed operationally.

## Company boundary expectations
Company context comes from token metadata. Request `companyId` must not override token ownership.

Resources outside the resolved company boundary should not be visible. Invisible resources may return `resource_not_found`.

Clients should not build flows that depend on cross-company identifiers.

## Request identifiers
Responses should include `requestId`.

Clients should log `requestId` safely and use it for support and investigation.

`requestId` is not a security credential and should not contain sensitive data.

## Idempotency for write operations
Payment creation should use `Idempotency-Key`.

Idempotency keys should be unique per logical operation and scoped by token, company, and endpoint.

Retrying the same request with the same key should be safe. Reusing the same key with a conflicting payload may return `idempotency_conflict`.

Conceptual request shape:
```text
POST /external/v1/payments
X-Api-Key: <api-key>
Idempotency-Key: payment-create-2026-05-01-001
```

## Rate limit handling
Clients should handle `429` responses.

Use `retryAfterSeconds` when provided. Otherwise apply exponential/backoff behavior.

Avoid tight retry loops. Payment status polling should be conservative.

Webhooks are preferred for asynchronous updates where available.

## Error handling
Clients should branch on stable error codes, not raw message text.

Status guidance:
- `401` means credential/token issue.
- `403` means authorization/scope/company-boundary issue.
- `404` can mean resource not visible under company boundary.
- `409` can mean idempotency conflict.
- `422` means request validation issue.
- `429` means throttling.
- `500`/`503` should be retried cautiously depending on idempotency.

| Error category | Example codes | Client action |
|---|---|---|
| Authentication/token | `authentication_failed`, `token_expired`, `token_revoked`, `token_suspended`, `token_not_active` | Fix or replace credentials before retrying. |
| Authorization/boundary | `insufficient_scope`, `endpoint_not_allowed`, `company_boundary_violation`, `resource_not_found` | Review scope, endpoint, resource ID, and company boundary expectations. |
| Validation | `validation_failed`, `invalid_request`, `unsupported_currency`, `unsupported_country` | Fix request fields before retrying. |
| Idempotency | `idempotency_key_required`, `idempotency_conflict` | Provide a key or resolve conflicting reuse. |
| Rate limiting | `rate_limit_exceeded` | Back off and retry later. |
| Webhook management | `invalid_webhook_url`, `webhook_target_rejected`, `webhook_event_type_not_supported`, `webhook_disabled` | Fix webhook configuration or status. |
| Server/platform | `internal_error`, `service_unavailable` | Retry cautiously with backoff depending on operation idempotency. |

## Webhook integration
Use the webhook registration endpoint. The webhook belongs to the resolved company.

Webhook URL must pass validation. Event payloads are minimal.

Verify signatures before processing. Deduplicate events by `eventId` or `deliveryId`.

Expect duplicate delivery. Do not expect exactly-once delivery.

Monitor delivery failures.

## Token lifecycle expectations
Raw token is shown only once. Token may expire, be suspended, or be revoked.

Clients should rotate before expiry. Unused credentials should be removed.

Compromised tokens should be reported and revoked. Revoked tokens should not be retried.

## Logging guidance for clients
- Log `requestId`.
- Log endpoint and status code.
- Log stable error code.
- Do not log raw API key.
- Avoid logging sensitive payment/account details.
- Avoid logging full webhook payloads unless policy allows.
- Secure operational logs.

## Operational checklist
- [ ] API key stored securely.
- [ ] Scopes reviewed and minimal.
- [ ] Sandbox and production keys separated.
- [ ] `requestId` captured in client logs.
- [ ] Write operations use idempotency.
- [ ] `429` retry handling implemented.
- [ ] Stable error codes handled.
- [ ] Webhook signatures verified.
- [ ] Webhook duplicate delivery handled.
- [ ] Token rotation process documented.
- [ ] Compromised token process documented.

## Common mistakes
- Storing API key in source code.
- Sending API key in query string.
- Logging raw API key.
- Assuming `companyId` in request controls access.
- Ignoring `insufficient_scope`.
- Retrying payment creation without idempotency.
- Tight polling of payment status.
- Ignoring `429` retry guidance.
- Processing unsigned webhooks.
- Assuming webhooks are exactly-once.
- Not rotating tokens before expiry.

## Related documents
- `docs/external-api-contract-examples.md`
- `docs/error-code-reference.md`
- `docs/webhook-delivery-model.md`
- `docs/token-lifecycle-and-revocation.md`
- `docs/rate-limiting-and-abuse-detection.md`
- `docs/scope-and-permission-model.md`
- `docs/company-scoped-api-key-model.md`

## Implementation notes
This guide is language-agnostic and conceptual. It does not define application code, framework boilerplate, SDKs, or an OpenAPI/YAML file.

Teams may adapt this guide into partner onboarding docs, API portal documentation, SDK guidance, or operational runbooks.
