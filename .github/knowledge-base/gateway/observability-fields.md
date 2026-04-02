# Observability Field Map — Checkout Gateway

**Phase**: 3 — Gateway & API Contracts
**Applies to**: All checkout and payment API requests, webhooks, and background jobs

---

## 1. Required Correlation Fields

Every request and background operation in the checkout and payment system must carry the following correlation fields. These fields must be propagated across service boundaries and included in every log entry, metric, and trace span.

| Field | Type | Source | Description |
|---|---|---|---|
| `request_id` | UUID v4 | Generated at API gateway on inbound request | Unique per HTTP request; echoed in `X-Request-ID` response header and `error.request_id` |
| `trace_id` | string (W3C traceparent) | Propagated from inbound `traceparent` header or generated if absent | Root trace identifier for distributed tracing |
| `session_id` | string | Created on session initiation (`POST /checkout/sessions`) | Links all requests within a checkout session |
| `order_id` | string | Provided by caller; created by backend | Links all events to a specific order |
| `payment_intent_id` | string | Created on `PaymentIntents.create()` | Links all payment events to a specific Stripe intent |

### Propagation Rules

- `request_id` is unique per HTTP request and must not be reused.
- `trace_id` must be forwarded to Stripe via the `X-B3-TraceId` header or W3C `traceparent` header so that Stripe support can correlate our traces with their logs.
- `session_id`, `order_id`, and `payment_intent_id` must be included as **structured fields** in every log entry after they are known. They must not be embedded only in message strings.
- Background jobs and webhook handlers must load these IDs from the event/job payload and attach them to the logging context before any processing.

---

## 2. Log Structure

All log entries must be structured JSON. Human-readable `message` fields are permitted but must be supplementary — all queryable data must be in discrete fields.

### 2.1 Standard Log Schema

```json
{
  "timestamp": "2024-01-15T14:00:00.000Z",
  "severity": "INFO",
  "service": "checkout-api",
  "environment": "production",
  "request_id": "req_abc123",
  "trace_id": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "session_id": "cs_live_abc123",
  "order_id": "ord_abc123",
  "payment_intent_id": "pi_3OxYz_abc123",
  "event": "payment_confirmed",
  "message": "Payment intent confirmed via webhook",
  "duration_ms": 42
}
```

| Field | Required | Notes |
|---|---|---|
| `timestamp` | Yes | ISO 8601 UTC |
| `severity` | Yes | `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL` |
| `service` | Yes | Service name (e.g., `checkout-api`, `payment-worker`) |
| `environment` | Yes | `development`, `staging`, `production` |
| `request_id` | Yes (if in request context) | Omit for background jobs without an originating request |
| `trace_id` | Yes | Always; generate if not propagated |
| `session_id` | When known | Include from the moment session is created |
| `order_id` | When known | Include from the moment order is created |
| `payment_intent_id` | When known | Include from the moment PaymentIntent is created |
| `event` | Yes | Machine-readable event name (see §3) |
| `message` | Yes | Human-readable description |
| `duration_ms` | On operations with measurable latency | Wall-clock time for the operation |

### 2.2 Prohibited Log Content

The following must **never** appear in any log entry:

- Raw card numbers, CVV, or expiry dates
- Stripe secret keys (`sk_live_*`, `sk_test_*`)
- Webhook signing secrets (`whsec_*`)
- Customer passwords or session tokens
- Bearer tokens or API keys
- Full HTTP request/response bodies unless body sanitisation is applied first

---

## 3. Event Catalogue

Each loggable event has a canonical `event` name used in structured logs, metrics, and dashboards.

### 3.1 Checkout Session Events

| Event Name | Severity | Description |
|---|---|---|
| `checkout.session.created` | INFO | New checkout session created |
| `checkout.session.retrieved` | DEBUG | Session fetched (poll or page load) |
| `checkout.session.expired` | INFO | Session reached expiry without payment |
| `checkout.session.cancelled` | INFO | Session explicitly cancelled |

### 3.2 Payment Events

| Event Name | Severity | Description |
|---|---|---|
| `payment.intent.created` | INFO | PaymentIntent created with Stripe |
| `payment.intent.confirmed` | INFO | Confirm request sent to Stripe |
| `payment.intent.requires_action` | INFO | 3DS or async payment action required |
| `payment.intent.processing` | INFO | Stripe reports processing state |
| `payment.intent.succeeded` | INFO | Payment captured successfully |
| `payment.intent.failed` | WARNING | Payment declined or failed |
| `payment.intent.cancelled` | INFO | Intent cancelled |
| `payment.refund.created` | INFO | Refund initiated |
| `payment.refund.succeeded` | INFO | Refund processed |
| `payment.refund.failed` | ERROR | Refund attempt failed |

### 3.3 Webhook Events

| Event Name | Severity | Description |
|---|---|---|
| `webhook.received` | DEBUG | Stripe webhook event received |
| `webhook.verified` | DEBUG | Signature verified successfully |
| `webhook.verification_failed` | ERROR | Signature verification failed — rejected |
| `webhook.duplicate` | INFO | Duplicate event ID — skipped |
| `webhook.ignored` | DEBUG | Unknown event type — accepted, not processed |
| `webhook.processed` | INFO | Event processed and state transitioned |
| `webhook.processing_error` | ERROR | Transient error during event processing |

### 3.4 Retry and Error Events

| Event Name | Severity | Description |
|---|---|---|
| `gateway.timeout` | WARNING | Stripe SDK timeout |
| `gateway.error` | ERROR | Stripe API 5xx error |
| `gateway.rate_limited` | WARNING | Stripe rate limit hit |
| `retry.attempt` | INFO | Retry attempt N of N |
| `retry.exhausted` | ERROR | All retry attempts failed |
| `reconciliation.started` | INFO | Reconciliation job started for ambiguous state |
| `reconciliation.resolved` | INFO | State resolved via reconciliation |
| `reconciliation.failed` | ERROR | Reconciliation could not resolve state |

---

## 4. Metrics

The following metrics must be emitted for the checkout and payment paths. Metrics must be tagged with `environment`, `service`, and `payment_method_type` at minimum.

### 4.1 Business Metrics

| Metric | Type | Tags | Description |
|---|---|---|---|
| `checkout.session.created.count` | Counter | env, method | Sessions created |
| `checkout.session.expired.count` | Counter | env, method | Sessions expired without payment |
| `payment.attempt.count` | Counter | env, method, status | Payment attempts by status |
| `payment.success.rate` | Gauge | env, method | Rolling success rate |
| `payment.decline.count` | Counter | env, method, decline_code | Declines by reason |
| `payment.refund.count` | Counter | env, method | Refunds issued |

### 4.2 Technical Metrics

| Metric | Type | Tags | Description |
|---|---|---|---|
| `api.request.duration_ms` | Histogram | env, endpoint, status_code | API endpoint latency |
| `gateway.call.duration_ms` | Histogram | env, operation | Stripe API call latency |
| `gateway.call.error.count` | Counter | env, operation, error_code | Stripe API errors |
| `webhook.processing.duration_ms` | Histogram | env, event_type | Webhook handler latency |
| `webhook.queue.depth` | Gauge | env | Unprocessed webhook events |
| `retry.count` | Counter | env, operation, attempt | Retry attempts |
| `reconciliation.count` | Counter | env, outcome | Reconciliation outcomes |

---

## 5. Distributed Trace Spans

Each meaningful unit of work must create a named trace span. Spans must be linked to the root `trace_id` via W3C traceparent propagation.

| Span Name | Parent | Key Attributes |
|---|---|---|
| `checkout.session.create` | Inbound request | `order_id`, `currency`, `amount` |
| `checkout.session.get` | Inbound request | `session_id` |
| `payment.intent.create` | `checkout.session.create` | `order_id`, `payment_method_types` |
| `payment.intent.confirm` | Inbound request | `payment_intent_id`, `payment_method_type` |
| `stripe.api.call` | Parent operation | `operation`, `stripe_request_id` |
| `webhook.handle` | Webhook receiver | `event_id`, `event_type`, `payment_intent_id` |
| `payment.state.transition` | Parent operation | `from_status`, `to_status`, `order_id` |
| `reconciliation.run` | Background scheduler | `payment_intent_id`, `expected_status` |

Span attributes must include the correlation fields listed in §1 whenever they are available.

---

## 6. Release Dashboard Requirements

The following telemetry fields and aggregations must be available in the release dashboard to gate launch and monitor production health.

### 6.1 Pre-Launch Gates (Go/No-Go)

| Signal | Threshold | Source |
|---|---|---|
| Payment success rate (credit card) | ≥ 95% | `payment.success.rate` metric |
| Payment success rate (Pix) | ≥ 98% | `payment.success.rate` metric |
| API P99 latency (confirm endpoint) | ≤ 3 000 ms | `api.request.duration_ms` histogram |
| Webhook processing P99 latency | ≤ 5 000 ms | `webhook.processing.duration_ms` histogram |
| `payment.refund.failed` rate | < 1% | Derived from `payment.refund.*` counters |
| `reconciliation.failed` in last 24h | 0 | `reconciliation.count` counter |

### 6.2 Ongoing Monitoring Alerts

| Alert | Condition | Severity |
|---|---|---|
| High decline rate | `payment.decline.count` increase > 20% vs 7-day baseline | Warning |
| Critical decline rate | `payment.decline.count` increase > 50% vs 7-day baseline | Critical |
| Gateway timeout spike | `gateway.timeout` > 5 in 5 minutes | Warning |
| Webhook verification failures | `webhook.verification_failed` > 0 | Critical |
| Retry exhaustion | `retry.exhausted` > 0 in 1 minute | Error |
| Reconciliation failures | `reconciliation.failed` > 0 | Error |
| API error rate | `api.request.duration_ms` with `status_code >= 500` > 1% of requests | Warning |

---

## 7. Logging Expectations by Payment Path

### 7.1 Successful Credit Card Payment

```
checkout.session.created       → INFO
payment.intent.created         → INFO
payment.intent.confirmed       → INFO
webhook.received               → DEBUG
webhook.verified               → DEBUG
payment.intent.succeeded       → INFO
webhook.processed              → INFO
payment.state.transition       → INFO  (processing → confirmed)
```

### 7.2 Declined Payment with Retry

```
checkout.session.created       → INFO
payment.intent.created         → INFO
payment.intent.confirmed       → INFO   (first attempt)
payment.intent.failed          → WARNING (decline_code=insufficient_funds)
webhook.received               → DEBUG
webhook.processed              → INFO
payment.state.transition       → INFO   (processing → failed)
--- user retries with different card ---
retry.attempt                  → INFO   (attempt 1 of 3)
payment.intent.confirmed       → INFO   (second attempt; new idempotency key)
payment.intent.succeeded       → INFO
payment.state.transition       → INFO   (failed → processing → confirmed)
```

### 7.3 Gateway Timeout with Reconciliation

```
payment.intent.confirmed       → INFO
gateway.timeout                → WARNING
retry.attempt                  → INFO   (attempt 1)
retry.attempt                  → INFO   (attempt 2)
retry.exhausted                → ERROR
reconciliation.started         → INFO
stripe.api.call (retrieve)     → INFO
reconciliation.resolved        → INFO
payment.state.transition       → INFO   (processing → confirmed)
```

### 7.4 Invalid Webhook Signature

```
webhook.received               → DEBUG
webhook.verification_failed    → ERROR  (includes request_id, remote IP)
[Request rejected — no further processing]
```
