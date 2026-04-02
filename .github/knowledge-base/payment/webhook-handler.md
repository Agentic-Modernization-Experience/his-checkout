# Webhook Handler — Stripe Event Ingestion and State Reconciliation

**Phase**: 4 — Backend Payment Orchestration  
**Provider**: Stripe

---

## 1. Ingestion Pipeline

```
Stripe ──HTTPS──► /webhooks/stripe (raw body)
                        │
                  [1] Signature Verification
                        │ fail → 400
                        │ pass
                  [2] Deduplication Check
                        │ duplicate → 200 (already_processed)
                        │ new
                  [3] Event Dispatch
                        │
                  [4] State Reconciliation
                        │
                  [5] Acknowledge → 200
```

---

## 2. Signature Verification

All incoming webhook requests **must** pass signature verification before any business logic executes. This is covered in full detail in `../security/webhook-verification.md`.

**Key requirements**:
- Use the official Stripe SDK `constructEvent()` method; do not reimplement.
- Pass the **raw** request body (bytes), not a parsed JSON object.
- Use `express.raw({ type: 'application/json' })` (or equivalent) middleware on the webhook endpoint.
- Reject with HTTP 400 on verification failure; log the failure with request metadata (no payload body).
- Each environment (`production`, `staging`, `development`) has its own `STRIPE_WEBHOOK_SECRET`.

---

## 3. Deduplication

After signature verification, check whether the event has already been processed:

```
1. Extract event.id (evt_xxx)
2. SELECT 1 FROM stripe_processed_events WHERE event_id = :eventId
3. If row exists → return HTTP 200 {"status": "already_processed"}
4. If not → INSERT INTO stripe_processed_events (event_id, received_at, event_type)
             (this INSERT happens within the same transaction as the state change in step 4)
5. Proceed to dispatch
```

The insert and state change **must** be atomic. If the state change transaction rolls back, the event ID must not remain in the deduplication store (otherwise the event will be dropped on retry).

---

## 4. Event Dispatch

### Handled Events

| Stripe Event | Handler | Action Summary |
|---|---|---|
| `payment_intent.succeeded` | `onPaymentSucceeded` | Mark attempt `captured`; transition order `→ confirmed` |
| `payment_intent.payment_failed` | `onPaymentFailed` | Mark attempt `failed`; record decline code; trigger retry or cancel |
| `payment_intent.requires_action` | `onPaymentRequiresAction` | Mark attempt `requires_action`; notify client to complete 3DS |
| `payment_intent.canceled` | `onPaymentCanceled` | Mark attempt `cancelled`; transition order `→ cancelled` (non-retryable) |
| `charge.refunded` | `onChargeRefunded` | Update `refund_records`; transition order state |
| `charge.dispute.created` | `onDisputeCreated` | Flag order `disputed`; freeze refund eligibility; alert operations |
| `charge.dispute.closed` | `onDisputeClosed` | Resolve dispute; restore order status if won; finalize refunded status if lost |

### Unhandled Events

Any event type not in the table above must be accepted with HTTP 200 and logged (event type only; no payload body logged). Do not return 4xx for unhandled event types — this causes Stripe to retry indefinitely.

```
onIgnoredEvent(event):
  log.info("stripe_webhook_ignored", { event_type: event.type, event_id: event.id })
  return HTTP 200 {"status": "ignored"}
```

---

## 5. Handler Implementations

### 5.1 `onPaymentSucceeded(event)`

```
Input: payment_intent.succeeded event
event.data.object = PaymentIntent

Steps:
1. Extract payment_intent_id from event.data.object.id
2. Load PaymentAttempt WHERE stripe_payment_intent_id = payment_intent_id
3. If not found: log warning ("orphaned_payment_intent"), return 200 (idempotent)
4. If attempt.status == 'captured': return 200 (already reconciled)
5. Extract charge_id from event.data.object.latest_charge
6. BEGIN TRANSACTION:
   a. UPDATE payment_attempts SET status='captured', stripe_charge_id=charge_id, updated_at=NOW()
      WHERE id = attempt.id AND status NOT IN ('captured', 'refunded')
   b. Load order WHERE id = attempt.order_id
   c. If order.status == 'pending_payment':
        UPDATE orders SET status='confirmed', updated_at=NOW(), version=version+1
        WHERE id = order.id AND version = order.version
   d. INSERT INTO stripe_processed_events (event_id, received_at, event_type)
7. COMMIT
8. Publish domain event: order.confirmed (via transactional outbox)
9. Return 200
```

### 5.2 `onPaymentFailed(event)`

```
Input: payment_intent.payment_failed event
event.data.object = PaymentIntent

Steps:
1. Extract payment_intent_id, last_payment_error from event
2. Load PaymentAttempt WHERE stripe_payment_intent_id = payment_intent_id
3. If not found or already 'failed': return 200 (idempotent)
4. Map last_payment_error.decline_code to internal failure_code (see decline code mapping)
5. BEGIN TRANSACTION:
   a. UPDATE payment_attempts SET status='failed', failure_code=:code, failure_message=:msg, updated_at=NOW()
      WHERE id = attempt.id AND (status = 'initiated' OR status = 'requires_action')
   b. Load order
   c. If order.attempt_count >= MAX_ATTEMPTS:
        UPDATE orders SET status='cancelled', updated_at=NOW(), version=version+1
        WHERE id = order.id AND version = order.version
      Else:
        order remains 'pending_payment' (client may retry)
   d. INSERT INTO stripe_processed_events (event_id, received_at, event_type)
6. COMMIT
7. If order cancelled: publish order.cancelled event (outbox)
8. Return 200
```

### 5.3 `onPaymentRequiresAction(event)`

```
Steps:
1. Load PaymentAttempt by payment_intent_id
2. If not found or terminal: return 200
3. BEGIN TRANSACTION:
   a. UPDATE payment_attempts SET status='requires_action', updated_at=NOW()
      WHERE id = attempt.id AND status = 'initiated'
   b. INSERT INTO stripe_processed_events
4. COMMIT
5. Publish notification to client channel (e.g., WebSocket / polling endpoint)
   with client_secret for 3DS confirmation
6. Return 200
```

### 5.4 `onPaymentCanceled(event)`

```
Steps:
1. Load PaymentAttempt by payment_intent_id
2. If not found or already 'cancelled': return 200
3. BEGIN TRANSACTION:
   a. UPDATE payment_attempts SET status='cancelled', updated_at=NOW()
      WHERE id = attempt.id AND status NOT IN ('captured', 'refunded', 'cancelled')
   b. Load order; if 'pending_payment':
        UPDATE orders SET status='cancelled', updated_at=NOW(), version=version+1
        WHERE id = order.id AND version = order.version
   c. INSERT INTO stripe_processed_events
4. COMMIT
5. Publish order.cancelled event (outbox)
6. Return 200
```

### 5.5 `onChargeRefunded(event)`

```
Input: charge.refunded event
event.data.object = Charge

Steps:
1. Extract charge_id, refund object(s) from event
2. Load PaymentAttempt WHERE stripe_charge_id = charge_id
3. If not found: log warning, return 200
4. For each new refund in event.data.object.refunds.data:
   a. Check if refund_records row exists for stripe_refund_id; skip if yes
   b. BEGIN TRANSACTION:
      - UPDATE refund_records SET status='succeeded', updated_at=NOW()
        WHERE stripe_refund_id = refund.id  (if row exists with 'pending')
        OR INSERT new row with status='succeeded' (if webhook arrives before initiateRefund response)
      - If all charge amount refunded: UPDATE payment_attempts SET status='refunded'
      - UPDATE orders SET status='refunded', version=version+1 (if fully refunded)
      - INSERT INTO stripe_processed_events
   c. COMMIT
5. Return 200
```

### 5.6 `onDisputeCreated(event)`

```
Steps:
1. Extract charge_id from event.data.object.charge
2. Load PaymentAttempt WHERE stripe_charge_id = charge_id
3. If not found: log warning, return 200
4. BEGIN TRANSACTION:
   a. UPDATE orders SET status='disputed', updated_at=NOW(), version=version+1
      WHERE id = attempt.order_id AND status NOT IN ('cancelled', 'refunded', 'disputed')
      AND version = order.version
   b. INSERT INTO stripe_processed_events
5. COMMIT
6. Publish alert: operations.dispute_alert (for manual review queue)
7. Return 200
```

---

## 6. Out-of-Order Event Handling

Stripe does not guarantee delivery order. The handlers above use current-state guards (e.g., `status NOT IN ('captured')`) to make every transition safe regardless of event arrival order.

**Key scenarios**:

| Scenario | Handling |
|---|---|
| `succeeded` arrives before `requires_action` | `succeeded` sets `captured`; subsequent `requires_action` is a no-op (status guard prevents regression) |
| `succeeded` arrives twice | Second call finds `status = captured`; returns 200 immediately |
| `failed` arrives after `captured` | Status guard `status = 'initiated' OR 'requires_action'` rejects the transition; 200 returned |
| Webhook for an order that was cancelled by timeout | Order already `cancelled`; handlers return 200 without re-opening order |

---

## 7. Stale Payment Intent Reconciliation

Orders in `pending_payment` for more than 30 minutes without a webhook event are candidates for reconciliation.

**Reconciliation job (scheduled every 5 minutes)**:

```
1. SELECT orders WHERE status = 'pending_payment' AND updated_at < NOW() - INTERVAL '30 minutes'
2. For each order:
   a. Load the active PaymentAttempt
   b. Call Stripe API: GET /v1/payment_intents/{payment_intent_id}
   c. Apply state based on Stripe status:
      - 'succeeded'           → trigger onPaymentSucceeded (synthetic event)
      - 'requires_payment_method' (failed) → trigger onPaymentFailed
      - 'canceled'            → trigger onPaymentCanceled
      - 'requires_action'     → trigger onPaymentRequiresAction
      - 'processing'          → no change; re-check next cycle
   d. If Stripe API unavailable (5xx/timeout): leave order in pending_payment;
      increment a reconciliation_attempt counter; alert ops if > 3 consecutive failures
3. If order has been in `pending_payment` for more than 30 minutes without a webhook event,
   the reconciliation job (step 1) already selected it. After all reconciliation attempts:
   → transition order to `cancelled` if Stripe status remains unresolvable after 3 consecutive failures; log for manual review
```

This reconciliation closes the gap between synchronous API errors, network failures, and webhook delivery delays.

---

## 8. Endpoint Configuration

| Property | Value |
|---|---|
| Path | `POST /webhooks/stripe` |
| Auth | None (signature-based via `STRIPE_WEBHOOK_SECRET`) |
| Body parser | `express.raw({ type: 'application/json' })` — raw bytes required |
| Rate limit | 100 req/min per IP |
| Protocol | HTTPS only |
| Timeout | 30 seconds (Stripe retries if no response within 30s) |
| Retry policy | Stripe retries for up to 3 days with exponential backoff on 5xx responses |
