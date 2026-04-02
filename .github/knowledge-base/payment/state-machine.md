# Payment State Machine — Backend Orchestration

**Phase**: 4 — Backend Payment Orchestration  
**Provider**: Stripe  
**Currency**: BRL

---

## 1. Order State Machine

### States

| State | Description | Terminal? |
|---|---|---|
| `draft` | Order created but not yet submitted | No |
| `pending_payment` | Order submitted; awaiting payment confirmation | No |
| `confirmed` | Payment captured; order accepted for fulfillment | No |
| `processing` | Order being prepared / inventory reserved | No |
| `completed` | Order fully fulfilled and delivered | Yes |
| `cancelled` | Order cancelled before fulfillment | Yes |
| `refunded` | Payment refunded (full or partial) | Yes |
| `disputed` | Chargeback raised; under dispute review | No |
| `manual_review` | Flagged for manual inspection (fraud, anomaly) | No |

### Transition Table

| From | Event | To | Guard / Invariant |
|---|---|---|---|
| `draft` | `submit_order` | `pending_payment` | Cart non-empty; amounts validated |
| `pending_payment` | `payment_intent.succeeded` (webhook) | `confirmed` | PaymentIntent ID matches order; `amount_cents` matches exactly (BRL is integer cents; no tolerance) |
| `pending_payment` | `payment_intent.payment_failed` (webhook) | `pending_payment` | Increment `attempt_count`; allow retry if `attempt_count < MAX_ATTEMPTS` |
| `pending_payment` | `payment_intent.payment_failed` — max attempts | `cancelled` | `attempt_count >= MAX_ATTEMPTS` (default 3) |
| `pending_payment` | `payment_intent.requires_action` (webhook) | `pending_payment` | Mark sub-state `requires_action`; client must complete 3DS |
| `pending_payment` | `payment_intent.canceled` (webhook) | `cancelled` | Terminal; no further retries |
| `pending_payment` | stale timeout (>30 min) | `cancelled` | Reconciliation job triggers cancellation |
| `confirmed` | `reserve_inventory` | `processing` | Inventory service confirms reservation |
| `confirmed` | `cancel_requested` | `cancelled` | Only before inventory reservation; triggers refund if charged |
| `processing` | `fulfillment_complete` | `completed` | Fulfillment service confirms delivery |
| `processing` | `cancel_requested` | `cancelled` | After inventory reserved; must release reservation and refund |
| `confirmed` | `charge.refunded` (webhook) | `refunded` | Stripe confirms refund; full or partial |
| `processing` | `charge.refunded` (webhook) | `refunded` | Stripe confirms refund; inventory released |
| `completed` | `charge.refunded` (webhook) | `refunded` | Post-fulfillment refund |
| `confirmed` | `charge.dispute.created` (webhook) | `disputed` | Freeze refund eligibility; flag for review |
| `processing` | `charge.dispute.created` (webhook) | `disputed` | Same as above |
| `disputed` | `charge.dispute.closed` (won) | Previous state (restored) | Dispute evidence accepted; pre-dispute status stored in order metadata |
| `disputed` | `charge.dispute.closed` (lost) | `refunded` | Chargeback accepted by bank |
| `pending_payment` | `flag_for_review` | `manual_review` | Fraud signal or anomaly detected |
| `manual_review` | `approve` | `confirmed` | Manual reviewer approves payment |
| `manual_review` | `reject` | `cancelled` | Manual reviewer rejects payment; triggers refund if charged |

### State Machine Diagram

```
                         submit_order
draft ──────────────────────────────────► pending_payment
                                                │
                    ┌───────────────────────────┼───────────────────────────────┐
                    │                           │                               │
            succeeded                    requires_action                  failed / timeout
                    │                     (3DS prompt)                          │
                    ▼                           │                               ▼
              confirmed ◄──────────────────────┘              (retry < MAX) → pending_payment
                    │                                          (retry >= MAX) → cancelled ──► [TERMINAL]
      ┌─────────────┼──────────────────────────────────┐
      │             │                                  │
 cancel_requested  reserve_inventory          charge.dispute.created
      │             │                                  │
      ▼             ▼                              disputed
  cancelled ──► processing                            │
  [TERMINAL]    │        │                   ┌────────┴────────┐
                │        │              won (restore)      lost
         fulfillment  cancel_requested       │                │
         _complete        │             (previous)        refunded
                │         ▼                           [TERMINAL]
                ▼     cancelled ──► [TERMINAL]
           completed ──► [TERMINAL]
                │
           charge.refunded
                │
                ▼
           refunded ──► [TERMINAL]
```

---

## 2. Payment Attempt State Machine

A payment attempt is created for each attempt to charge a payment method. It is scoped to an order and provider reference.

### States

| State | Description | Terminal? |
|---|---|---|
| `initiated` | PaymentIntent created; client confirmation pending | No |
| `requires_action` | 3DS or additional authentication required | No |
| `processing` | Stripe is processing the charge | No |
| `captured` | Charge succeeded; funds captured | Yes |
| `failed` | Charge failed (decline, error) | Yes |
| `cancelled` | PaymentIntent cancelled | Yes |
| `refunded` | Captured amount refunded (full or partial) | Yes |

### Transition Table

| From | Event | To | Notes |
|---|---|---|---|
| `initiated` | Stripe confirmation pending | `requires_action` | 3DS flow |
| `initiated` | Stripe confirms immediately | `captured` | Non-3DS success |
| `initiated` | Stripe declines | `failed` | Decline codes logged |
| `requires_action` | Customer completes 3DS | `captured` | Via `payment_intent.succeeded` |
| `requires_action` | 3DS fails or times out | `failed` | Via `payment_intent.payment_failed` |
| `requires_action` | Cancelled before completion | `cancelled` | Via `payment_intent.canceled` |
| `processing` | `payment_intent.succeeded` | `captured` | Async confirmation |
| `processing` | `payment_intent.payment_failed` | `failed` | Async failure |
| `captured` | `charge.refunded` | `refunded` | Triggered by refund operation |

---

## 3. Invariants

The following invariants must be enforced by the service layer and persistence model:

1. **Single active attempt**: An order may have at most one non-terminal payment attempt at a time.
2. **Monotonic attempt count**: `attempt_count` only increases; never decreases.
3. **Amount immutability**: The `amount_cents` and `currency` of an order cannot change after `pending_payment`.
4. **Idempotent transitions**: Applying the same event to the same state must produce the same result and not create duplicate records.
5. **No backward transitions**: State transitions follow the defined graph; backward moves are rejected.
6. **Terminal state immutability**: Orders in `completed`, `cancelled`, `refunded` cannot be transitioned except via explicit authorized compensations.
7. **PaymentIntent binding**: Each payment attempt is bound to exactly one Stripe PaymentIntent ID; that binding cannot change after creation.
8. **Webhook authority**: Stripe webhook events are the authoritative source for payment state; synchronous API responses are confirmatory, not authoritative.

---

## 4. MAX_ATTEMPTS Policy

| Parameter | Value | Notes |
|---|---|---|
| `MAX_ATTEMPTS` | 3 | Maximum payment retries per order |
| Retry eligibility | `payment_intent.payment_failed` only | `canceled` events are non-retryable |
| Cooldown between retries | None required (client-driven) | Backend enforces count, not timing |
| Reset policy | Attempts never reset; count is per-order lifetime | Prevents unlimited retries via order re-creation |
