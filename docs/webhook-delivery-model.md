# Webhook delivery model

## Purpose
Webhooks allow external clients to receive company-scoped event notifications without polling.

The webhook delivery model covers:
- event delivery,
- company-scoped ownership,
- webhook endpoint registration,
- delivery signing,
- retries,
- idempotency,
- failure handling,
- auditability.

Webhooks do not replace API authentication, token lifecycle controls, audit logging, or explicit external contracts. They are an event delivery mechanism that must still follow the same company boundary, security, and operational expectations as the rest of the external API surface.

## Why webhook delivery matters in fintech external APIs
Payment status changes are often asynchronous. A client may create a payment and need to learn later when the status changes.

Excessive polling can create unnecessary load. Webhooks reduce repeated status requests while still giving clients timely updates.

External clients need reliable event notifications, but webhook targets are outside platform control. Clients can misconfigure endpoints, rotate certificates, change network rules, or fail to process events correctly.

Event delivery must not leak data across companies. Each event must be owned by one company context, and delivery must only go to webhooks registered for that company.

Failed delivery must be observable and recoverable. Operations teams need to understand whether a delivery was attempted, succeeded, retried, failed, or disabled.

Webhook abuse can create operational risk. High registration churn, unsafe targets, or repeated delivery failures can consume platform capacity and complicate incident response.

## Design goals
- **Company-scoped event delivery**: events are delivered only to webhooks owned by the event's company.
- **Explicit webhook registration**: clients register endpoints through a documented external contract.
- **Signed delivery payloads**: receivers can verify that the delivery came from the platform.
- **Stable event identifiers**: clients can deduplicate event processing.
- **Idempotent receiver behavior**: duplicate deliveries do not create duplicate business effects.
- **Bounded retries**: delivery attempts are limited and observable.
- **Observable delivery outcomes**: attempts, success, failure, disablement, and recovery are traceable.
- **Minimal event payloads**: events include enough context to act, not full sensitive resource payloads.
- **Clear failure handling**: repeated failures lead to defined operational outcomes.
- **Separation between webhook registration and event delivery**: management endpoints and outbound delivery have distinct controls.

## Non-goals
- Replacing the notification system.
- Guaranteeing exactly-once delivery.
- Defining a full event streaming platform.
- Defining vendor-specific queue infrastructure.
- Storing sensitive full payloads in delivery logs.
- Providing production-ready code.

## Core concepts
- **Webhook subscription**: a company-owned registration that connects event types to a client endpoint.
- **Webhook endpoint**: the external HTTPS URL that receives deliveries.
- **Event type**: a documented external event name, such as `payment.status_changed`.
- **Event ID**: stable identifier for the event itself.
- **Delivery ID**: identifier for one delivery attempt or delivery workflow.
- **Signing secret**: secret or equivalent verification material used to sign delivery payloads.
- **Delivery attempt**: one outbound attempt to send an event to a webhook endpoint.
- **Retry policy**: bounded rules for retrying transient delivery failures.
- **Dead-letter handling**: controlled handling for events that cannot be delivered after retries.
- **Company boundary**: the resolved company ownership that determines which webhooks may receive an event.
- **Minimal payload**: event payload that contains only necessary fields and references.

## Webhook registration model
Each webhook belongs to exactly one company. A webhook is created under the resolved token owner's company, and caller-provided `companyId` must not define ownership.

Registered event types must be explicit. URL validation is required before activation. Webhook management requires the `webhooks:manage` scope, and webhook lifecycle changes should be audited.

Conceptual registration fields:
| Field | Purpose | Required? | Notes |
|---|---|---|---|
| `webhookId` | Stable webhook registration identifier. | yes | Used for management, audit, and delivery correlation. |
| `companyId` | Owning company for the webhook. | yes | Resolved from token ownership at registration time. |
| `url` | External delivery endpoint. | yes | Must pass target validation before activation. |
| `eventTypes` | External event types subscribed by the webhook. | yes | Use documented external event names. |
| `status` | Current webhook lifecycle status. | yes | Examples include `active`, `disabled`, or `deleted`. |
| `createdAt` | Registration timestamp. | yes | Supports audit and support review. |
| `updatedAt` | Last registration update timestamp. | yes | Supports lifecycle review. |
| `signingSecretFingerprint` | Non-secret reference to signing material. | yes | The signing secret itself must not be logged. |
| `lastDeliveryAt` | Last attempted or successful delivery timestamp. | no | Define the exact update policy. |
| `failureCount` | Recent or current failure count. | no | Supports disablement and operations review. |
| `metadata` | Controlled non-sensitive extension fields. | no | Must not contain secrets or sensitive payloads. |

## Webhook target validation
- Require HTTPS.
- Reject localhost/private network targets unless explicitly controlled by platform policy.
- Validate URL format.
- Avoid internal service callback targets.
- Consider domain ownership or verification where appropriate.
- Avoid embedding secrets in callback URLs.
- Audit validation failures.

## Event payload model
Webhook payloads should be minimal and company-scoped.

Conceptual payload:
```json
{
  "eventId": "evt_001",
  "eventType": "payment.status_changed",
  "occurredAt": "2026-05-01T12:00:00Z",
  "companyId": "company_12",
  "resourceType": "payment",
  "resourceId": "pay_001",
  "data": {
    "status": "processing"
  }
}
```

`eventId` is stable. `companyId` is resolved from platform ownership. The payload should not include unnecessary sensitive data.

Clients can call external API endpoints for details if authorized. Payload schema should be versioned or documented so clients can process events without depending on internal event structures.

## Delivery signing model
Each webhook should have a signing secret or equivalent verification mechanism. Delivery payloads should be signed, and the signature should cover timestamp and body.

The receiver should verify the signature before processing. The signing secret should not be logged, and secret rotation should be supported.

Conceptual delivery headers:
- `X-Webhook-Event-Id`
- `X-Webhook-Delivery-Id`
- `X-Webhook-Timestamp`
- `X-Webhook-Signature`
- `X-Webhook-Retry-Count`

## Delivery idempotency
The same event may be delivered more than once. Receivers should deduplicate by `eventId` or `deliveryId` depending on use case.

The platform should use a stable `eventId`. `deliveryId` identifies one delivery attempt or delivery workflow.

Retry delivery must not create duplicate business effects. Exactly-once delivery should not be promised.

## Retry and backoff model
Retry only transient failures. Use bounded retry attempts and increasing backoff.

Avoid infinite retry loops. Do not retry permanent validation failures indefinitely. Track delivery attempts and expose delivery status to operations if applicable.

Conceptual retry states:
| State | Meaning | Next action |
|---|---|---|
| `pending` | Event is queued or ready for delivery. | Attempt delivery. |
| `delivered` | Receiver accepted the event. | No further delivery action. |
| `retrying` | Prior attempt failed with a retryable outcome. | Retry within bounded policy. |
| `failed` | Retry policy is exhausted or failure is permanent. | Record failure and consider dead-letter handling. |
| `disabled` | Webhook is not eligible for delivery. | Require explicit review or re-enable action. |

## Failure handling
Failed webhook delivery should be observable. Repeated failures may disable a webhook temporarily, and the integration owner may need to be contacted.

Dead-letter handling may be used when events cannot be delivered after bounded retries. Failed delivery should not expose event data in unsafe logs.

Failure should produce audit or operational events so teams can review the webhook, event type, delivery attempt, reason, and next action.

## Audit expectations
Audit or operational events should cover:
- webhook created,
- webhook updated,
- webhook disabled,
- webhook re-enabled,
- webhook deleted,
- delivery attempted,
- delivery succeeded,
- delivery failed,
- repeated failure threshold reached,
- signing secret rotated.

Events should include:
- `webhookId`,
- `companyId`,
- `eventType`,
- `eventId`,
- `deliveryId`,
- `decision` or `outcome`,
- `reason`,
- `requestId` or `correlationId` where available.

## Rate limiting and abuse considerations
Webhook registration should be strictly rate-limited. Delivery retries should be bounded.

Repeated failing targets should not consume unbounded capacity. High webhook churn is an abuse signal.

Delivery failure spikes should be monitored. Webhook events should remain company-scoped.

## Relationship with external API contracts
The webhook registration endpoint must be explicit. Event schemas must be documented.

Error semantics for webhook management should be stable. Webhook delivery payloads are part of the external product surface.

Clients should not rely on internal event names.

## Relationship with threat model
Unsafe webhook targets can cause data leakage. Unsigned payloads can be spoofed.

Excessive retries can create denial-of-service risk. Large payloads can increase sensitive data exposure.

Receiver deduplication failures can create duplicated business effects.

## Common mistakes
- Sending unsigned webhook payloads.
- Including too much sensitive data in event payloads.
- Allowing localhost/private network webhook targets without policy.
- Retrying forever.
- Promising exactly-once delivery.
- Not documenting event schemas.
- Using internal event names directly.
- Not auditing webhook management actions.
- Not rotating signing secrets.
- Not rate-limiting webhook registration.

## Implementation notes
This model is language-agnostic and conceptual. It does not define application code.

Implementations may use queues, event buses, delivery workers, retry schedulers, signing services, and audit events depending on platform architecture. The model does not prescribe a vendor or framework.
