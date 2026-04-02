# Webhook Verification Policy — Stripe Events

**Phase**: 2 — Security & Compliance  
**Provider**: Stripe

---

## 1. Policy Statement

All Stripe webhook events **must** be verified before any payment state mutation occurs. Unverified events must be rejected immediately with HTTP 400. No business logic may execute on an unverified payload.

---

## 2. Signature Verification

Stripe signs every webhook delivery with an HMAC-SHA256 signature included in the `Stripe-Signature` header.

### Verification Algorithm

```
1. Extract the raw request body (bytes, before any parsing)
2. Extract the `Stripe-Signature` header
3. Call stripe.webhooks.constructEvent(rawBody, signatureHeader, STRIPE_WEBHOOK_SECRET)
4. If the call throws → reject (HTTP 400), log the failure, do not process
5. If the call succeeds → proceed to deduplication check
```

### Implementation Requirements

- **Use the official Stripe SDK** `constructEvent()` method — do not reimplement signature verification manually.
- **Pass the raw request body**, not a parsed JSON object. Parsing before verification breaks the signature.
- The `STRIPE_WEBHOOK_SECRET` used must match the secret registered for the specific endpoint in the Stripe Dashboard.
- The timestamp tolerance window is **300 seconds** (5 minutes) — Stripe SDK default. Do not increase this window.

### Example (Node.js)

```javascript
app.post('/webhooks/stripe', express.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.headers['stripe-signature'];
  let event;
  try {
    event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET);
  } catch (err) {
    // Verification failed — reject
    return res.status(400).send(`Webhook verification failed: ${err.message}`);
  }
  // Proceed to deduplication and event handling
});
```

> The endpoint **must** use `express.raw()` (or equivalent), not `express.json()`, so the raw body bytes are preserved for signature validation.

---

## 3. Event Deduplication

Stripe may deliver the same event more than once (at-least-once delivery guarantee). The backend must be idempotent with respect to duplicate events.

### Deduplication Store

- Maintain a persistent store (database table or cache with TTL) keyed by Stripe event ID (`evt_xxx`).
- Before processing any event, check whether the event ID has already been seen.
- If already processed: return HTTP 200 immediately without re-processing.
- If not seen: insert the event ID (with timestamp), then process.

### Deduplication Table Schema (reference)

```sql
CREATE TABLE stripe_processed_events (
  event_id    VARCHAR(64)  PRIMARY KEY,   -- evt_xxx from Stripe
  received_at TIMESTAMPTZ  NOT NULL,
  event_type  VARCHAR(128) NOT NULL
);

-- Index to support efficient time-based purge queries
CREATE INDEX idx_stripe_events_received_at ON stripe_processed_events (received_at);
```

### TTL / Retention

- Event IDs should be retained for **30 days**.
- Events older than 30 days may be purged; Stripe does not re-deliver events after that window.
- Purge strategy: a scheduled job (e.g., daily cron) runs `DELETE FROM stripe_processed_events WHERE received_at < NOW() - INTERVAL '30 days'`. The `received_at` index ensures this query does not require a full table scan.

---

## 4. Accepted Events and Ordering

### Events Handled

| Stripe Event | Action | Notes |
|---|---|---|
| `payment_intent.succeeded` | Transition order to `confirmed`; mark payment `captured` | Primary success path |
| `payment_intent.payment_failed` | Mark payment `failed`; trigger recovery flow | Includes decline and authentication failure |
| `payment_intent.requires_action` | Mark payment `requires_action`; prompt client to complete 3DS | 3DS challenge path |
| `payment_intent.canceled` | Mark order `cancelled` | Stripe-initiated or backend-initiated cancellation |
| `charge.refunded` | Transition payment to `refunded` (full or partial) | Post-capture refund |
| `charge.dispute.created` | Flag order for dispute review; freeze refund eligibility | Chargeback path |
| `charge.dispute.closed` | Resolve dispute outcome; restore order if won, finalize refund if lost | Dispute resolution |

### Events Explicitly Ignored

Any event type not in the list above must be accepted (HTTP 200) but not acted upon. This prevents Stripe from retrying unhandled events indefinitely.

### Ordering Assumptions

- Event delivery order is **not guaranteed** by Stripe. The backend must handle out-of-order events safely.
- State transitions must be guarded by current-state checks (e.g., only transition `pending_payment → confirmed` if order is currently in `pending_payment`).
- Duplicate events are idempotent by construction (deduplication store).

---

## 5. Response Requirements

| Scenario | HTTP Response | Body |
|---|---|---|
| Signature verification failed | 400 | Short error message (no internal detail) |
| Duplicate event (already processed) | 200 | Empty or `{"status":"already_processed"}` |
| Event type not handled | 200 | Empty or `{"status":"ignored"}` |
| Event processed successfully | 200 | Empty or `{"status":"ok"}` |
| Processing error (transient) | 500 | Stripe will retry |

> Returning 200 for unhandled events is required to stop Stripe retry attempts. **Do not** return 4xx for unknown event types.

---

## 6. Endpoint Security

- The webhook endpoint must not require session authentication (Stripe calls it server-to-server).
- The `STRIPE_WEBHOOK_SECRET` provides the authentication mechanism.
- The endpoint should rate-limit to prevent exhaustion attacks (e.g., 100 req/min per IP).
- Webhook URL must be HTTPS. Stripe will not deliver to plain HTTP endpoints in production.
- Webhook URL must not be publicly documented or easy to guess; obscurity is not the control but reduces noise.

---

## 7. Verification Checklist

- [ ] Webhook endpoint uses raw body middleware (not JSON-parsed body)
- [ ] `constructEvent()` is called before any business logic
- [ ] Events failing verification are rejected with HTTP 400
- [ ] Deduplication store is checked before processing each event
- [ ] Each environment has its own `STRIPE_WEBHOOK_SECRET`
- [ ] Unhandled event types return HTTP 200 without processing
- [ ] Timestamp tolerance is not extended beyond 300 seconds
- [ ] Webhook endpoint is served over HTTPS only
