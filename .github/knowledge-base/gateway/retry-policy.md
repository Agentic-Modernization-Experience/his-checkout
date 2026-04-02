# Retry Policy — Checkout Gateway Requests

**Phase**: 3 — Gateway & API Contracts
**Provider**: Stripe
**Applies to**: Outbound calls to Stripe, inbound retries from clients, and internal compensation logic

---

## 1. Policy Overview

All mutation requests (creating sessions, confirming payments) must use idempotency keys so that retries are safe. Reads are inherently idempotent and may be retried freely.

The retry policy is split into three domains:

| Domain | Retry Owner | Mechanism |
|---|---|---|
| **Client → Backend** | Frontend client | Idempotency key + bounded retry with backoff |
| **Backend → Stripe** | Backend gateway service | Stripe SDK retry + idempotency key |
| **Stripe → Backend** (webhooks) | Stripe (at-least-once delivery) | Deduplication store (see `webhook-verification.md`) |

---

## 2. Client → Backend Retry Policy

### 2.1 Retryable Conditions

A client may retry a request if and only if:

| HTTP Status | `error.code` | Retryable |
|---|---|---|
| 429 | `rate_limited` | Yes — after `Retry-After` seconds |
| 500 | `internal_error` | Yes — after backoff |
| 502 | `gateway_error` | Yes — after backoff |
| 503 | `service_unavailable` | Yes — after `Retry-After` seconds |
| 504 | `gateway_timeout` | Yes — after backoff |
| All 4xx except above | — | **No** — do not retry; user/input error |

Network-level failures (TCP reset, connection refused, no response) are retryable.

### 2.2 Retry Parameters

| Parameter | Value | Notes |
|---|---|---|
| Max retry attempts | 3 | Not counting the original request |
| Initial backoff | 1 second | Before first retry |
| Backoff multiplier | 2× (exponential) | 1s → 2s → 4s |
| Jitter | ±20% of backoff interval | Prevents thundering herd |
| Max backoff cap | 10 seconds | Never wait longer than 10s between retries |
| Total retry window | ~15 seconds | Cumulative across all retry attempts |

**Backoff Schedule (example)**

| Attempt | Wait Before Attempt | Cumulative Time |
|---|---|---|
| 1 (original) | 0s | 0s |
| 2 (retry 1) | ~1s | ~1s |
| 3 (retry 2) | ~2s | ~3s |
| 4 (retry 3) | ~4s | ~7s |

### 2.3 Idempotency Key Reuse on Retry

- The **same** idempotency key must be used across all retry attempts for the same logical operation.
- A new idempotency key must only be generated for a **new** user action (e.g., user clicks "Try Again" after a final failure).
- See §5 below for idempotency key format rules.

---

## 3. Backend → Stripe Retry Policy

### 3.1 Stripe SDK Configuration

All Stripe API calls from the backend must use the official Stripe SDK with the following settings:

| Setting | Value | Rationale |
|---|---|---|
| `maxNetworkRetries` | 2 | SDK-level retries on connection errors and 500/503 |
| `timeout` | 30 000 ms (30s) | Per-request timeout; Stripe P99 is well under this |
| Idempotency key | Always set on write operations | Stripe uses this for deduplication on their side |

```javascript
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  maxNetworkRetries: 2,
  timeout: 30000,
});
```

### 3.2 Operations That Must Carry an Idempotency Key

| Stripe API Call | Idempotency Key Derivation |
|---|---|
| `PaymentIntents.create()` | `pi-create-{order_id}` |
| `PaymentIntents.confirm()` | `pi-confirm-{order_id}-{attempt_number}` |
| `PaymentIntents.cancel()` | `pi-cancel-{order_id}` |
| `Refunds.create()` | `refund-{order_id}-{refund_reason}` |
| `PaymentIntents.capture()` | `pi-capture-{order_id}` |

The `attempt_number` in `confirm` keys increments when the user explicitly retries (e.g., tries a different card). The key must remain stable across network-level retries of the same confirm call.

### 3.3 Stripe Error Handling

| Stripe Error Type | Action |
|---|---|
| `StripeConnectionError` | Retry with SDK `maxNetworkRetries`; map to `gateway_timeout` if all retries fail |
| `StripeAPIError` (5xx) | Retry with SDK; map to `gateway_error` if all retries fail |
| `StripeCardError` | Do **not** retry; map decline code and return `402` to client |
| `StripeInvalidRequestError` | Do **not** retry; indicates a programming error; log and return `500` |
| `StripeAuthenticationError` | Do **not** retry; alert on-call; return `500` to client |
| `StripeRateLimitError` | Retry after `Retry-After` header delay; map to `gateway_error` if exhausted |
| `StripeIdempotencyError` | Do **not** retry; request payload mismatch; return `409 idempotency_conflict` |

---

## 4. Timeout Policy

### 4.1 End-to-End Timeout Budget

```
Client request timeout    60 seconds   (frontend hard limit)
Backend API timeout       45 seconds   (Nginx / load balancer)
Backend → Stripe timeout  30 seconds   (Stripe SDK per-call)
Stripe P99 latency        ~5 seconds   (Stripe SLA reference)
```

The 30-second Stripe timeout is intentionally conservative. The backend timeout (45s) is longer to accommodate SDK retries: up to 2 SDK retries × ~5s latency + overhead ≈ 15–20s worst case, leaving headroom.

### 4.2 Timeout Behaviour

| Layer | On Timeout | Client-Visible Effect |
|---|---|---|
| Stripe SDK | StripeConnectionError → SDK retry → `gateway_timeout` | `504` with `error.code: gateway_timeout` |
| Backend API | Returns 504 to client | Client receives `gateway_timeout` |
| Frontend | Shows retry modal | User can retry with same idempotency key |

**Important**: A timeout does **not** mean the payment failed. Stripe may have processed the PaymentIntent successfully before the timeout was detected. The backend must reconcile state via the webhook event rather than assuming failure on timeout.

---

## 5. Idempotency Key Behavior

### 5.1 Client-Provided Idempotency Keys

All mutation endpoints require an `Idempotency-Key` request header.

| Requirement | Rule |
|---|---|
| Format | UUID v4 (`xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`) |
| Length | 36 characters |
| Uniqueness scope | Per user session + operation pair |
| Lifetime | 24 hours — backend caches responses for this duration |
| Case sensitivity | Treated as case-sensitive; always use lowercase |

### 5.2 Key Generation Rules

- **Frontend**: Generate a fresh UUID at the moment the user initiates an action (e.g., page load for session creation, button click for payment confirm).
- **Retry**: Reuse the same UUID generated for the original attempt.
- **New action after failure**: Generate a new UUID (represents user explicitly choosing to retry or try a different method).

### 5.3 Cached Response Behaviour

When a request with a previously seen idempotency key is received within 24 hours:

1. Backend verifies that the new request payload matches the original.
2. If payload **matches**: return the cached response with header `X-Idempotency-Replayed: true`. No re-processing.
3. If payload **differs**: return `409 idempotency_conflict` immediately.

### 5.4 Endpoints Requiring Idempotency Key

| Endpoint | Required |
|---|---|
| `POST /api/v1/checkout/sessions` | Yes |
| `POST /api/v1/payments/intents/{id}/confirm` | Yes |
| `DELETE /api/v1/checkout/sessions/{id}` | No (idempotent by nature) |
| `GET` endpoints | No |
| `POST /api/v1/webhooks/stripe` | No (Stripe sends its own event IDs) |

---

## 6. Pix and Boleto Polling Policy

Pix and Boleto payments do not complete synchronously. The frontend must poll for status updates.

### 6.1 Pix Polling

| Parameter | Value |
|---|---|
| Poll endpoint | `GET /api/v1/payments/intents/{id}` |
| Poll interval | Every 5 seconds |
| Max poll duration | 30 minutes (Pix expiry) |
| Stop polling on | `status` becomes `confirmed`, `failed`, or `cancelled` |

### 6.2 Boleto Polling

| Parameter | Value |
|---|---|
| Poll endpoint | `GET /api/v1/payments/intents/{id}` |
| Poll interval | Every 60 seconds |
| Max poll duration | 3 business days (boleto expiry) |
| Stop polling on | `status` becomes `confirmed`, `cancelled`, or `expired` |

Clients must use exponential back-off for poll retries on network errors (same schedule as §2.2). Polling requests do not require an idempotency key.

---

## 7. Compensation and Fallback on Ambiguous State

When the backend cannot determine whether a payment succeeded (timeout, network partition, missing webhook), the following fallback procedure applies:

```
1. Log the ambiguous event with payment_intent_id and correlation fields
2. Mark the internal payment status as `processing` (not `failed`)
3. Schedule a reconciliation job to poll Stripe directly:
   stripe.paymentIntents.retrieve(payment_intent_id)
4. Apply the Stripe status mapping (§4.3 Canonical Status Lifecycle in api-contracts.md) to resolve the state
5. If Stripe confirms `succeeded` → transition to `confirmed`, notify customer
6. If Stripe confirms `canceled`/`failed` → transition to `failed`, notify customer
7. If Stripe API is also unreachable after 3 poll attempts (backoff 5s/30s/120s):
   - Alert on-call with full context
   - Leave order in `pending_payment`; do not auto-cancel
   - Surface a "payment pending" message to the customer
```

Never auto-cancel or auto-fail a payment based solely on a timeout. The canonical state is always the Stripe-confirmed state delivered via webhook or explicit polling.
