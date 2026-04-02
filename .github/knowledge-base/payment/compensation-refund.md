# Compensation and Refund Logic — Cancellation, Refund, and Recovery

**Phase**: 4 — Backend Payment Orchestration  
**Provider**: Stripe  
**Currency**: BRL

---

## 1. Overview

Compensation logic handles the reversal of completed or in-progress payment operations. Every compensation action must be:
- **Idempotent**: repeatable without side effects if already completed.
- **Traceable**: logged with actor ID, reason, and outcome.
- **Webhook-confirmed**: final state is set only upon Stripe webhook confirmation, not on the synchronous API response.

---

## 2. Cancellation

### 2.1 Pre-Payment Cancellation (Order in `draft` or `pending_payment`)

When no charge has been captured, cancellation is straightforward:

```
1. Assert order.status ∈ {draft, pending_payment}
2. If a PaymentAttempt exists in status 'initiated' or 'requires_action':
   a. Call Stripe API: POST /v1/payment_intents/{pi_xxx}/cancel
      with idempotency key: 'cancel-{order_id}-{attempt_id}'
   b. The 'payment_intent.canceled' webhook will confirm the cancellation
3. Transition order → cancelled (immediately; do not wait for webhook in pre-capture case)
4. Log: { actor_id, order_id, reason, timestamp }
```

**Note**: For `draft` orders with no PaymentIntent, simply mark as `cancelled` — no Stripe call needed.

### 2.2 Post-Capture Cancellation (Order in `confirmed` or `processing`)

When a charge has already been captured, cancellation requires a refund:

```
1. Assert order.status ∈ {confirmed, processing}
2. Assert a PaymentAttempt with status = 'captured' exists
3. Call initiateRefund(orderId, full_amount, reason='cancellation')
4. Do NOT transition order to 'cancelled' immediately
5. Order transitions to 'refunded' only upon charge.refunded webhook confirmation
6. Log cancellation intent: { actor_id, order_id, reason, refund_initiated: true }
```

### 2.3 Cancellation After Fulfillment (Order in `completed`)

Post-fulfillment cancellations are handled as returns, not cancellations. Apply the same refund flow and mark order `refunded`. The inventory and fulfillment reversal is handled by the fulfillment service receiving the `order.refunded` domain event.

---

## 3. Refund Operations

### 3.1 Full Refund

```
Input: orderId, reason
Idempotency key: 'refund-{orderId}-full-{reason}'

1. Load order; assert status ∈ {confirmed, processing, completed}
2. Load PaymentAttempt; assert status = 'captured'
3. Check idempotency store (return existing if key present)
4. Call Stripe API:
   POST /v1/refunds
   { charge: charge_id, reason: stripe_reason_code, metadata: { order_id, initiated_by } }
   with idempotency key
5. Persist RefundRecord { stripe_refund_id, amount_cents: full, status: 'pending' }
6. Store idempotency key
7. Return RefundRecord (status will be updated to 'succeeded' via charge.refunded webhook)
```

### 3.2 Partial Refund

```
Input: orderId, amountCents, reason
Idempotency key: 'refund-{orderId}-{amountCents}-{reason}'

Steps identical to full refund except:
- Pass amount parameter to Stripe API: { charge, amount: amountCents, reason }
- Validate: amountCents <= (captured_amount - already_refunded_amount)
- Order status remains unchanged after partial refund (not moved to 'refunded')
- Only full refund or cumulative total reaching captured amount triggers order → 'refunded'
```

### 3.3 Dispute-Initiated Refund

When a chargeback is resolved against the merchant (`dispute_resolved_lost`):

```
1. No Stripe API call needed — bank forces the refund
2. charge.refunded webhook is received from Stripe
3. onChargeRefunded handler processes it normally
4. Update dispute record with outcome = 'lost'
5. Order transitions to 'refunded'
6. Alert operations team
```

---

## 4. Stripe Reason Code Mapping

| Internal Reason | Stripe Reason Code | Notes |
|---|---|---|
| `cancellation` | `requested_by_customer` | Order cancelled before or after fulfillment |
| `duplicate_order` | `duplicate` | Duplicate detected by idempotency system |
| `fraud` | `fraudulent` | Confirmed fraud or dispute lost |
| `item_not_received` | `requested_by_customer` | Fulfillment failure reported by customer |
| `defective_item` | `requested_by_customer` | Quality issue |
| `chargeback` | (not applicable — bank-initiated) | Outcome of `charge.dispute.created` flow |

Only these Stripe reason codes are valid: `duplicate`, `fraudulent`, `requested_by_customer`. All internal reasons must map to one of these.

---

## 5. Inventory Reservation and Release

### Reservation Timing

Inventory is reserved **after** payment confirmation, not before. This avoids reserving stock that may never convert.

```
Order flow with inventory:
1. Order submitted → pending_payment (no reservation)
2. payment_intent.succeeded webhook → order confirmed
3. Domain event: order.confirmed published (transactional outbox)
4. Inventory service receives order.confirmed → reserves stock
5. Inventory service publishes: inventory.reserved
6. Order service receives inventory.reserved → order → processing
7. Fulfillment begins
```

### Release Timing

Inventory is released in the following scenarios:

| Trigger | Reservation Release |
|---|---|
| Order cancelled before confirmation | No release needed (inventory was never reserved) |
| Order cancelled after confirmation but before `inventory.reserved` | Release not yet needed; cancellation event handled by inventory service upon receiving `order.cancelled` |
| Order cancelled after `inventory.reserved` | Release triggered by `order.cancelled` domain event |
| Refund after fulfillment | Return/return-to-stock handled by fulfillment service on `order.refunded` |

---

## 6. Recovery Hooks

### 6.1 Failed Webhook Delivery Recovery

If Stripe cannot reach the webhook endpoint (5xx or timeout), it retries with exponential backoff for up to 3 days. No additional action is required from the backend for normal webhook failures. However, if the endpoint is unreachable for an extended period, use the reconciliation job in `webhook-handler.md` section 7 to recover state proactively.

### 6.2 Partial Refund Failure Recovery

If a refund API call to Stripe fails (network error, Stripe 5xx):

```
1. Persist a failed RefundRecord attempt: { status: 'failed', error_code }
2. The idempotency key remains valid
3. A retry can be submitted with the same idempotency key (safe to repeat)
4. If 3 consecutive failures: escalate to operations queue for manual investigation
5. The order remains in its current status — do not transition on API failure
```

### 6.3 Orphaned PaymentIntent Recovery

If a PaymentIntent was created but no webhook was received and the order is stale:

```
See webhook-handler.md §7 — Stale Payment Intent Reconciliation
```

### 6.4 Dispute Recovery (Won)

If a dispute is resolved in the merchant's favour, Stripe sends a `charge.dispute.closed` event with `dispute.status = 'won'`:

```
1. Stripe sends charge.dispute.closed with dispute.status='won'
2. Handler: verify dispute status is 'won'; unset 'disputed' flag
3. Restore order to previous status (before dispute): use the pre-dispute status stored in order metadata
4. Log: { dispute_id, outcome: 'won', order_id, restored_status }
```

---

## 7. Compensation Audit Log

All compensation actions (cancellations, refunds, releases) must emit a structured log entry:

```json
{
  "event": "compensation_action",
  "action": "refund_initiated | order_cancelled | inventory_released",
  "order_id": "<uuid>",
  "payment_attempt_id": "<uuid>",
  "stripe_refund_id": "<re_xxx or null>",
  "amount_cents": 12500,
  "currency": "BRL",
  "reason": "cancellation",
  "actor_id": "<user_id or 'system'>",
  "timestamp": "<ISO 8601>",
  "trace_id": "<distributed trace ID>"
}
```

This log is written to the application event stream and must be retained for 13 months (per `data-classification.md`).

---

## 8. Compensation Checklist

- [ ] Cancellation before capture requires no refund; marks order `cancelled` immediately
- [ ] Cancellation after capture must call `initiateRefund()` first; order moves to `refunded` on webhook
- [ ] All refund calls use idempotency keys (safe to retry)
- [ ] Stripe refund reason codes are mapped from internal reasons before the API call
- [ ] Inventory reservation happens only after `order.confirmed` domain event
- [ ] Inventory release happens on `order.cancelled` or `order.refunded` domain events (handled by inventory service)
- [ ] All compensation actions produce an audit log entry
- [ ] Dispute-forced refunds are handled via `charge.refunded` webhook (no additional API call needed)
- [ ] Post-fulfillment refunds are treated as returns; fulfillment reversal is notified via `order.refunded`
