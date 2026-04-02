# Service Layer — Checkout-to-Payment Orchestration

**Phase**: 4 — Backend Payment Orchestration  
**Provider**: Stripe  
**Currency**: BRL

---

## 1. Service Overview

The orchestration layer sits between the checkout API and the payment provider. It coordinates order creation, payment intent lifecycle, webhook reconciliation, and compensation actions. All operations are idempotent and traceable.

```
Client
  │
  ▼
CheckoutController          (HTTP layer — validates input, routes to service)
  │
  ▼
OrderService                (creates/reads orders; enforces state invariants)
  │         │
  │         ▼
  │    PaymentOrchestrationService    (coordinates payment intent lifecycle)
  │         │         │
  │         │         ▼
  │         │    StripeGatewayAdapter   (Stripe SDK wrapper; idempotency keys)
  │         │
  │         ▼
  │    PaymentAttemptRepository        (persists attempts; idempotency store)
  │
  ▼
OrderRepository             (persists orders; optimistic locking)
```

---

## 2. Core Operations

### 2.1 `createOrder(request)`

**Responsibility**: Validate and persist an order in `draft` state.

**Inputs**: cart items, customer ID, billing/shipping address, currency  
**Outputs**: `Order` with `id`, `status: draft`, `amount_cents`, `idempotency_key`

**Idempotency**: keyed on `customer_id + cart_hash + timestamp_bucket` (per-session key generated client-side and sent as header `Idempotency-Key`).

**Steps**:
1. Check idempotency store — if key exists, return stored response.
2. Validate cart amounts against current catalog prices.
3. Persist order with `status = draft`.
4. Store idempotency key with response reference.
5. Return order.

---

### 2.2 `initiatePayment(orderId, paymentMethodToken, idempotencyKey)`

**Responsibility**: Transition order to `pending_payment` and create a Stripe PaymentIntent.

**Inputs**: `order_id`, `pm_xxx` token, client-supplied `Idempotency-Key`  
**Outputs**: `PaymentAttempt` with `stripe_payment_intent_id`, `client_secret`

**Idempotency**: The `Idempotency-Key` header is forwarded to the Stripe API call verbatim. If the same key is replayed, the existing PaymentIntent is returned.

**Steps**:
1. Load order; assert `status ∈ {draft, pending_payment}` (reject `pending_payment` only if `attempt_count >= MAX_ATTEMPTS`).
   - `draft`: first payment attempt
   - `pending_payment` with `attempt_count < MAX_ATTEMPTS`: retry after a previous failed attempt
   - Any other status: reject with `INVALID_STATE`
2. Check idempotency store — return existing attempt if key present.
3. Assert `attempt_count < MAX_ATTEMPTS` (reject with `MAX_ATTEMPTS_EXCEEDED` if not).
4. Transition order `draft → pending_payment` (atomic with attempt creation).
5. Create Stripe PaymentIntent via `StripeGatewayAdapter.createPaymentIntent()`.
6. Persist `PaymentAttempt` with `status = initiated`, `stripe_payment_intent_id`.
7. Increment `attempt_count` on order.
8. Store idempotency key.
9. Return `PaymentAttempt` with `client_secret` for client-side confirmation.

**Error handling**:
- Stripe API error on creation: mark attempt `failed`; do not transition order state; allow retry.
- Network timeout creating PaymentIntent: check existing intents via idempotency key before creating a new one.

---

### 2.3 `confirmPayment(orderId, paymentIntentId)`

**Responsibility**: Reconcile a synchronous payment confirmation against the authoritative state from webhook events.

**Note**: Synchronous confirmation from the client is treated as a hint only. The order is **not** marked `confirmed` until a `payment_intent.succeeded` webhook is received and verified. This service call registers the intent and polls/waits for webhook reconciliation.

**Steps**:
1. Load order; assert `status = pending_payment`.
2. Load `PaymentAttempt` by `stripe_payment_intent_id`; assert it belongs to the order.
3. If attempt `status = captured` (webhook already processed): transition order `pending_payment → confirmed` and return.
4. If attempt `status = initiated` or `requires_action`: register pending reconciliation record; return status `awaiting_webhook`.
5. If attempt `status = failed` or `cancelled`: return error with current state.

---

### 2.4 `handleWebhookEvent(stripeEvent)`

**Responsibility**: Process a verified Stripe event and apply the correct state transition.

**See also**: `webhook-handler.md` for ingestion and verification details.

**Dispatch table**:

| Event Type | Handler Method |
|---|---|
| `payment_intent.succeeded` | `onPaymentSucceeded(event)` |
| `payment_intent.payment_failed` | `onPaymentFailed(event)` |
| `payment_intent.requires_action` | `onPaymentRequiresAction(event)` |
| `payment_intent.canceled` | `onPaymentCanceled(event)` |
| `charge.refunded` | `onChargeRefunded(event)` |
| `charge.dispute.created` | `onDisputeCreated(event)` |
| (any other) | `onIgnoredEvent(event)` — log and return |

---

### 2.5 `cancelOrder(orderId, reason, actorId)`

**Responsibility**: Cancel an order and trigger a refund if a charge exists.

**Steps**:
1. Load order; assert cancellable states: `draft`, `pending_payment`, `confirmed`, `processing`.
2. If `confirmed` or `processing` and a captured payment attempt exists: call `initiateRefund()`.
3. Transition order → `cancelled`.
4. If inventory was reserved: publish `inventory.release` event.
5. Log cancellation with `reason` and `actorId`.

---

### 2.6 `initiateRefund(orderId, amountCents?, reason)`

**Responsibility**: Refund a captured payment, fully or partially.

**Idempotency**: keyed on `order_id + refund_reason + amount_cents`.

**Steps**:
1. Load order; assert `status ∈ {confirmed, processing, completed}`.
2. Load captured `PaymentAttempt`; assert `status = captured`.
3. Check idempotency store.
4. Call `StripeGatewayAdapter.createRefund(chargeId, amountCents)`.
5. Persist `RefundRecord` with Stripe refund ID.
6. Do not transition order state here — wait for `charge.refunded` webhook.

---

## 3. Idempotency Key Design

All write operations accept an `Idempotency-Key` header. Keys are:
- Client-generated UUIDs (v4) for user-facing operations.
- System-generated deterministic keys for internal operations (e.g., `cancel-{orderId}-{timestamp_minute}`).

**Storage**:
```sql
CREATE TABLE idempotency_keys (
  key           VARCHAR(128) PRIMARY KEY,
  operation     VARCHAR(64)  NOT NULL,         -- e.g. 'initiate_payment'
  entity_id     VARCHAR(64)  NOT NULL,         -- order_id or attempt_id
  response_body JSONB,                          -- cached response
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  expires_at    TIMESTAMPTZ  NOT NULL           -- created_at + 30 days
);

CREATE INDEX idx_idempotency_keys_expires ON idempotency_keys (expires_at);
```

Keys expire after 30 days (matching Stripe's idempotency window). A scheduled purge job runs daily.

---

## 4. Error Response Contract

| Error Code | HTTP Status | Meaning |
|---|---|---|
| `INVALID_STATE` | 409 | Order not in required state for this operation |
| `MAX_ATTEMPTS_EXCEEDED` | 422 | Payment retry limit reached |
| `PAYMENT_INTENT_NOT_FOUND` | 404 | No matching attempt for the given PaymentIntent ID |
| `REFUND_NOT_ELIGIBLE` | 422 | Order not in a refundable state |
| `IDEMPOTENT_REPLAY` | 200 | Duplicate request; returns cached response |
| `PROVIDER_ERROR` | 502 | Stripe API returned an unexpected error |
| `INVALID_SIGNATURE` | 400 | Webhook signature verification failed |

Raw Stripe error codes and messages are **never** forwarded to clients. All errors are mapped to the above contract before returning.
