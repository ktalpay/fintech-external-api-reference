# ADR-0007: Use signed webhook delivery model

- **Status**: Accepted
- **Date**: 2026-05-01

## Context

External fintech clients often need to react to asynchronous events such as payment status changes. Polling can create unnecessary load and makes client behavior harder to control.

Webhook delivery improves integration usability, but it introduces outbound delivery risks. Webhook targets are outside platform control, payloads can be spoofed if unsigned, retries can create operational load, and event data must remain company-scoped and minimal.

## Decision

Use a signed webhook delivery model for company-scoped external events. Webhook subscriptions belong to exactly one company, event payloads are minimal and company-scoped, deliveries include stable event and delivery identifiers, retry behavior is bounded, and delivery outcomes are auditable.

Webhook registration and delivery will be treated as separate concerns. Registration requires explicit management scope and target validation. Delivery requires signing, documented event schemas, bounded retries, and observable outcomes.

## Consequences

Positive:

- Reduces polling pressure.
- Improves external integration usability.
- Supports asynchronous payment/status workflows.
- Improves spoofing resistance through signing.
- Makes delivery failures easier to investigate.

Trade-offs:

- Requires signing secret lifecycle management.
- Requires retry/failure handling ownership.
- Requires event schema documentation.
- Requires careful payload minimization.
- Requires operational handling for failed or disabled webhooks.

## Alternatives considered

1. **Polling-only model**
   - Rejected: polling can create unnecessary load and slower client reaction to asynchronous changes.

2. **Unsigned webhook payloads**
   - Rejected: receivers need a way to verify that deliveries came from the platform.

3. **Unbounded retries**
   - Rejected: repeated failures could consume capacity and hide integration defects.

4. **Full-resource webhook payloads**
   - Rejected: large payloads increase sensitive data exposure and make schemas harder to stabilize.

5. **Sharing internal event names directly**
   - Rejected: external event names should be stable product contracts rather than internal implementation details.
