# Fraud Control Checklist — Checkout Payments

**Phase**: 2 — Security & Compliance  
**Fraud Provider**: None (Stripe Radar built-in rules only)

---

## 1. Fraud Control Approach

This checkout implementation relies on **Stripe Radar** (included with Stripe) for automated fraud detection. No additional third-party fraud provider is configured at this phase. All fraud signals are evaluated by Stripe before the PaymentIntent transitions to `succeeded` or `requires_action`.

---

## 2. Stripe Radar Built-In Controls

The following Radar capabilities are active by default with no additional configuration:

| Control | Description | Default State |
|---|---|---|
| Machine learning risk scoring | Radar evaluates each charge against global fraud signals | Active |
| Card testing detection | Repeated low-value authorization attempts blocked | Active |
| Block high-risk cards | Cards flagged in Stripe's global network blocked | Active |
| 3DS authentication trigger | Radar triggers 3DS for elevated-risk charges | Active |
| CVC verification | CVC mismatch results in decline | Active |
| Postal code / ZIP verification | Address verification signal used in scoring | Active |

---

## 3. Velocity and Abuse Controls

These controls must be implemented at the application layer, regardless of Stripe Radar:

### 3.1 Client-Side Controls

| Control | Implementation |
|---|---|
| Disable submit button after first click | Prevent accidental double-submission from the same session |
| Show loading state during payment processing | Prevent impatient re-submission |
| Block re-submission until server response received | Enforced via UI state management |

### 3.2 Backend Rate Limits

| Endpoint | Limit | Window | Action on Breach |
|---|---|---|---|
| `POST /checkout/sessions` | 10 requests | Per user per hour | HTTP 429, log event |
| `POST /payments/confirm` | 5 requests | Per order ID | HTTP 429, log event |
| `POST /webhooks/stripe` | 100 requests | Per IP per minute | HTTP 429 |

### 3.3 Idempotency as Fraud Control

- Every payment confirmation request carries a unique idempotency key (see `idempotency-keys` section).
- Retrying with the same key returns the original result; it cannot create a new charge.
- This prevents duplicate charges from network retries, refreshes, or client bugs.

---

## 4. Anti-Duplication Controls

| Scenario | Control |
|---|---|
| Same user submits twice (double-click) | Idempotency key bound to `order_id + attempt_number`; second call returns first result |
| Network timeout causes client to retry | Same idempotency key reused; Stripe returns original charge outcome |
| Session resume on page reload | Backend checks order status before allowing re-confirm; completed orders are not recharged |
| Duplicate Stripe webhook | Deduplication store keyed by `event_id` prevents double state transition |
| Race condition (concurrent requests) | Order state machine uses optimistic locking or database-level transaction guards |

---

## 5. Audit Logging Requirements

All payment-adjacent events must be logged with enough detail to support incident investigation and chargeback defense.

### Required Log Fields

| Field | Description |
|---|---|
| `event_type` | e.g., `payment.initiated`, `payment.confirmed`, `payment.failed`, `webhook.received` |
| `order_id` | First-party order identifier |
| `payment_intent_id` | Stripe `pi_xxx` |
| `user_id` | Authenticated user (if available) |
| `ip_address` | Masked (last octet removed) |
| `timestamp` | ISO 8601 UTC |
| `outcome` | Result or error code (no raw Stripe error messages forwarded to logs in plain text) |
| `idempotency_key` | The key used for this request |

### Prohibited Log Content

- PAN, CVV, expiry date, cardholder name
- Raw Stripe secret key values
- Full IP address (use masked form)
- Raw internal stack traces in production logs (map to error codes)

---

## 6. Incident Handling — Suspicious Payment Activity

### Detection Signals

The following should trigger immediate review:

- Authorization rate drops below 80% over a 15-minute window
- More than 10 failed `payment.failed` events from the same IP or user in 10 minutes
- Duplicate `payment.confirmed` events for the same order ID
- Webhook signature verification failures exceeding 5 per minute
- Any `charge.dispute.created` event (chargeback)

### Response Runbook (Summary)

| Severity | Trigger | Response |
|---|---|---|
| Low | Isolated declined charge | Log only; no action |
| Medium | Velocity breach (rate limit hit) | Block IP temporarily; alert on-call |
| High | Multiple disputes or confirmed fraud pattern | Freeze affected payment methods; alert on-call; escalate to Stripe |
| Critical | Active card testing or credential stuffing attack | Enable Stripe Radar blocking rules; temporary checkout throttle; security team notified |

### Chargeback Response

1. Log the dispute event (`charge.dispute.created`).
2. Freeze any pending refunds or captures related to the disputed order.
3. Gather evidence: order record, IP log, user agent, delivery confirmation.
4. Submit evidence to Stripe Dashboard within the dispute deadline.
5. Document the outcome and update fraud pattern rules if applicable.

---

## 7. Future Fraud Controls (Post-Phase-2)

The following controls are deferred and should be revisited when transaction volume justifies them:

| Control | When to Consider |
|---|---|
| Dedicated fraud platform (e.g., Stripe Radar Rules, Sift, Kount) | When dispute rate exceeds 0.5% |
| Device fingerprinting | When card testing events appear in Stripe logs |
| Behavioral biometrics | When authorized fraud (account takeover) is detected |
| Manual review queue | When automated decline rate produces significant false positives |

---

## 8. Summary Checklist

- [ ] Stripe Radar is enabled on the Stripe account (default)
- [ ] CVC and AVS verification signals are sent to Stripe on each charge
- [ ] 3DS is configured to trigger for Radar high-risk charges
- [ ] Submit button is disabled after first click (client-side)
- [ ] Rate limits are enforced on checkout and payment endpoints (backend)
- [ ] Idempotency keys are generated and persisted per payment attempt
- [ ] Deduplication store is in place for webhook events
- [ ] Audit log captures all required fields without sensitive data
- [ ] Alert thresholds are configured for authorization rate drop and velocity breach
- [ ] Chargeback response runbook is documented and accessible to on-call team
