# Reliability and Operational Safety — Payment Orchestration

**Phase**: 4 — Backend Payment Orchestration  
**Provider**: Stripe  
**Currency**: BRL

---

## 1. Fallback Behavior — Provider Unavailability

When Stripe is unavailable or returns inconsistent responses, the backend must degrade gracefully without corrupting order state.

### Provider Status Matrix

| Stripe Response | Backend Behavior | Order State |
|---|---|---|
| HTTP 200 (success) | Process normally | Transitions per state machine |
| HTTP 429 (rate limit) | Retry with exponential backoff (see §2) | Unchanged until retry succeeds or fails |
| HTTP 500 / 503 | Log error; do not change order state; allow retry on next client attempt | Unchanged |
| Network timeout on PaymentIntent creation | Check existing intent by idempotency key before creating new one | Unchanged |
| Network timeout on refund creation | Retry with same idempotency key | Unchanged |
| Stripe status API unavailable during reconciliation | Skip cycle; increment consecutive-failure counter; alert ops if > 3 | Unchanged |
| Stripe webhook delivery failure | Stripe retries for 3 days; reconciliation job covers the gap | Covered by reconciliation job |

### Consistency Principle

The order state **must never** be advanced based on an uncertain or missing provider response. State transitions require:
- A verified webhook event (preferred, authoritative), OR
- A confirmed synchronous API response AND subsequent reconciliation confirmation.

---

## 2. Retry Controls

### Stripe API Retries (Backend → Stripe)

| Scenario | Retry Strategy |
|---|---|
| HTTP 429 (rate limit) | Exponential backoff: 1s, 2s, 4s, 8s, 16s — max 5 attempts |
| HTTP 500 / 503 | Exponential backoff: 2s, 4s, 8s — max 3 attempts |
| Network timeout | Single immediate retry; then fail |
| HTTP 400 (invalid request) | No retry (client error; fix the request) |
| HTTP 401 / 403 | No retry; alert ops (key rotation may be needed) |

All retries use the **same idempotency key** to prevent duplicate charges or refunds.

### Client Payment Retries (Client → Backend)

| Scenario | Retry Eligibility |
|---|---|
| Payment attempt `failed` (decline) | Eligible if `attempt_count < MAX_ATTEMPTS` (default 3) |
| Payment attempt `cancelled` | Not eligible; order is cancelled |
| Payment attempt `requires_action` | Not a failure; client must complete 3DS challenge |
| Order `cancelled` | Not eligible; new order must be created |

### Webhook Retries (Stripe → Backend)

- Stripe retries on HTTP 5xx or no response within 30 seconds.
- Retry schedule: approximately 5m, 30m, 2h, 5h, 10h, ... up to 3 days.
- The backend webhook handler is idempotent; Stripe retries are safe.
- Do not artificially delay webhook responses. Process within 10 seconds when possible.

---

## 3. Dead-Letter Handling

### Definition

An event or operation enters dead-letter state when it has exhausted all retry attempts and cannot be processed automatically.

### Dead-Letter Queue Candidates

| Source | Dead-Letter Trigger | Action |
|---|---|---|
| Reconciliation job | Order stuck in `pending_payment` > 30 min AND Stripe status unresolvable after 3 attempts | Cancel order; publish `ops.dead_letter_alert` |
| Refund retry | 3 consecutive refund API failures for the same order | Publish `ops.refund_failed_alert`; order stays in current state |
| Webhook processing error | Unhandled exception in webhook handler after all retries | Log full event details (redacted); publish `ops.webhook_processing_failed` |
| Outbox relay | Outbox event not delivered after 5 attempts | Mark outbox event as dead; publish `ops.outbox_dead_letter` |

### Dead-Letter Alert Payload

```json
{
  "alert": "dead_letter",
  "source": "reconciliation_job | refund_retry | webhook_handler | outbox_relay",
  "order_id": "<uuid>",
  "payment_attempt_id": "<uuid or null>",
  "event_id": "<evt_xxx or null>",
  "failure_reason": "<short description>",
  "attempt_count": 3,
  "last_error": "<safe error summary; no raw provider messages>",
  "timestamp": "<ISO 8601>",
  "trace_id": "<distributed trace ID>"
}
```

### Dead-Letter Resolution

Dead-letter alerts are sent to an operations queue for manual investigation. The on-call engineer must:
1. Review the alert payload and trace ID.
2. Determine if the issue is transient (infrastructure) or permanent (data integrity).
3. Either re-trigger the operation manually or update the order state directly with an audit log entry.
4. Mark the dead-letter record as resolved with outcome and actor ID.

---

## 4. Logging and Tracing Fields

All payment-domain log entries must include the following structured fields to support cross-service investigations.

### Required Fields (All Payment Events)

| Field | Type | Description |
|---|---|---|
| `trace_id` | string | Distributed trace ID (propagated via HTTP header `X-Trace-ID` or `traceparent`) |
| `span_id` | string | Current span ID within the trace |
| `order_id` | UUID | The affected order |
| `customer_id` | string | Masked customer identifier (never log raw email or name) |
| `event` | string | Structured event name (see event catalog below) |
| `timestamp` | ISO 8601 | UTC timestamp of the log entry |
| `severity` | string | `DEBUG`, `INFO`, `WARN`, `ERROR` |

### Conditional Fields (Operation-Specific)

| Field | Present When |
|---|---|
| `payment_attempt_id` | Any payment attempt operation |
| `stripe_payment_intent_id` | PaymentIntent created or updated |
| `stripe_charge_id` | Charge captured or refunded |
| `stripe_refund_id` | Refund operation |
| `stripe_event_id` | Webhook event received |
| `stripe_event_type` | Webhook event received |
| `from_status` | Any state transition |
| `to_status` | Any state transition |
| `failure_code` | Payment failed |
| `attempt_number` | Payment retry |
| `actor_id` | Manual or system-initiated action |
| `idempotency_key` | Write operations (masked: first 8 chars + `***`) |

### Event Catalog

| Event Name | Severity | Trigger |
|---|---|---|
| `order_created` | INFO | New order persisted |
| `payment_initiated` | INFO | PaymentIntent created |
| `payment_succeeded` | INFO | `payment_intent.succeeded` webhook processed |
| `payment_failed` | WARN | `payment_intent.payment_failed` webhook processed |
| `payment_requires_action` | INFO | 3DS challenge required |
| `payment_cancelled` | INFO | PaymentIntent cancelled |
| `order_confirmed` | INFO | Order transitioned to confirmed |
| `order_cancelled` | INFO | Order cancelled |
| `refund_initiated` | INFO | Refund API call made |
| `refund_succeeded` | INFO | `charge.refunded` webhook processed |
| `refund_failed` | ERROR | Refund API call failed after retries |
| `dispute_created` | WARN | `charge.dispute.created` webhook processed |
| `webhook_verification_failed` | WARN | Stripe signature check failed |
| `webhook_duplicate` | DEBUG | Already-processed event ID received |
| `webhook_ignored` | DEBUG | Unhandled event type received |
| `reconciliation_triggered` | INFO | Stale order reconciliation job ran |
| `dead_letter_alert` | ERROR | Operation exhausted retries |
| `idempotent_replay` | DEBUG | Duplicate idempotency key detected |

### Fields Never Logged

The following must **never** appear in logs under any circumstances:

- Raw card data (PAN, CVV, expiry, cardholder name)
- `STRIPE_SECRET_KEY` or `STRIPE_WEBHOOK_SECRET`
- Full `pm_xxx` tokens (mask to show only prefix and last 4: `pm_***1234`)
- Raw Stripe API error objects (map to safe internal codes first)
- Full customer email or name (mask per `data-classification.md`)
- Full billing/shipping addresses

---

## 5. Observability Checklist

- [ ] All log entries include `trace_id`, `order_id`, `event`, and `timestamp`
- [ ] State transitions log `from_status` and `to_status`
- [ ] Webhook events log `stripe_event_id` and `stripe_event_type`
- [ ] Failures log `failure_code` (internal mapping; not raw Stripe code)
- [ ] Dead-letter alerts are routed to the operations queue with full context
- [ ] No PII or secrets appear in logs
- [ ] Distributed trace context (`traceparent` or `X-Trace-ID`) is propagated to Stripe API calls and logged
- [ ] Retry attempts are logged with attempt number and backoff duration
- [ ] Idempotency key replays are logged at DEBUG level
