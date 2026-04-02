# Persistence Model — Payment Orchestration

**Phase**: 4 — Backend Payment Orchestration  
**Provider**: Stripe  
**Currency**: BRL

---

## 1. Schema Overview

```
orders
  └── payment_attempts  (1:N — one per retry)
        └── refund_records  (1:N — one per refund action)
idempotency_keys          (deduplication for write operations)
stripe_processed_events   (webhook deduplication; see webhook-handler.md)
```

---

## 2. `orders` Table

Authoritative record of each customer order. State is the single source of truth for order lifecycle.

```sql
CREATE TABLE orders (
  id                 UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id        VARCHAR(64)  NOT NULL,
  status             VARCHAR(32)  NOT NULL DEFAULT 'draft',  -- see state-machine.md
  amount_cents       BIGINT       NOT NULL,                  -- immutable after pending_payment
  currency           CHAR(3)      NOT NULL DEFAULT 'BRL',
  attempt_count      SMALLINT     NOT NULL DEFAULT 0,
  idempotency_key    VARCHAR(128) UNIQUE,                    -- client-supplied key for createOrder
  metadata           JSONB,                                  -- cart snapshot, shipping address, etc.
  created_at         TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at         TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  version            BIGINT       NOT NULL DEFAULT 0         -- optimistic locking counter
);

CREATE INDEX idx_orders_customer_id  ON orders (customer_id);
CREATE INDEX idx_orders_status       ON orders (status);
CREATE INDEX idx_orders_created_at   ON orders (created_at);
```

### Optimistic Locking

All `UPDATE` statements on `orders` must include a `WHERE version = :currentVersion` clause and increment `version`:

```sql
UPDATE orders
SET status = :newStatus, updated_at = NOW(), version = version + 1
WHERE id = :orderId AND version = :currentVersion;
```

If the update affects 0 rows, a concurrent modification occurred — retry the operation or return a conflict error.

---

## 3. `payment_attempts` Table

Each row represents one attempt to charge a payment method for an order.

```sql
CREATE TABLE payment_attempts (
  id                        UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id                  UUID         NOT NULL REFERENCES orders(id),
  attempt_number            SMALLINT     NOT NULL,               -- 1-indexed; matches orders.attempt_count
  status                    VARCHAR(32)  NOT NULL DEFAULT 'initiated',
  stripe_payment_intent_id  VARCHAR(64)  UNIQUE NOT NULL,        -- pi_xxx; immutable after creation
  stripe_payment_method_id  VARCHAR(64),                         -- pm_xxx; stored for audit purposes only; never reused without explicit customer consent
  stripe_charge_id          VARCHAR(64),                         -- ch_xxx; populated on capture
  amount_cents              BIGINT       NOT NULL,
  currency                  CHAR(3)      NOT NULL DEFAULT 'BRL',
  failure_code              VARCHAR(64),                         -- Stripe decline code if failed
  failure_message           VARCHAR(256),                        -- mapped (safe) failure description
  idempotency_key           VARCHAR(128) UNIQUE,                 -- key used when creating the PaymentIntent
  metadata                  JSONB,                               -- Stripe metadata passed to PaymentIntent
  created_at                TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at                TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_payment_attempts_order_attempt
  ON payment_attempts (order_id, attempt_number);

CREATE INDEX idx_payment_attempts_order_id
  ON payment_attempts (order_id);

CREATE INDEX idx_payment_attempts_intent_id
  ON payment_attempts (stripe_payment_intent_id);
```

### Notes

- `stripe_payment_method_id` is stored for audit purposes only; it is never re-used without explicit customer consent.
- `failure_code` and `failure_message` are mapped from Stripe's decline codes to safe internal labels. Raw Stripe messages are not stored.
- `stripe_charge_id` is populated when the `payment_intent.succeeded` webhook is received.

---

## 4. `refund_records` Table

Each row represents one refund action against a captured payment attempt.

```sql
CREATE TABLE refund_records (
  id                    UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
  payment_attempt_id    UUID         NOT NULL REFERENCES payment_attempts(id),
  order_id              UUID         NOT NULL REFERENCES orders(id),
  stripe_refund_id      VARCHAR(64)  UNIQUE NOT NULL,   -- re_xxx from Stripe
  amount_cents          BIGINT       NOT NULL,           -- amount refunded (may be partial)
  currency              CHAR(3)      NOT NULL DEFAULT 'BRL',
  reason                VARCHAR(64)  NOT NULL,           -- 'duplicate', 'fraudulent', 'requested_by_customer', 'cancellation'
  status                VARCHAR(32)  NOT NULL DEFAULT 'pending',  -- pending, succeeded, failed
  initiated_by          VARCHAR(64)  NOT NULL,           -- actor_id: user, system, or 'stripe_dispute'
  idempotency_key       VARCHAR(128) UNIQUE,
  created_at            TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_refund_records_order_id
  ON refund_records (order_id);

CREATE INDEX idx_refund_records_attempt_id
  ON refund_records (payment_attempt_id);
```

---

## 5. `idempotency_keys` Table

Deduplication store for all mutating service operations.

```sql
CREATE TABLE idempotency_keys (
  key           VARCHAR(128) PRIMARY KEY,
  operation     VARCHAR(64)  NOT NULL,        -- e.g. 'initiate_payment', 'create_order', 'initiate_refund'
  entity_id     VARCHAR(64)  NOT NULL,        -- order_id or attempt_id the operation acted on
  response_body JSONB,                         -- serialized response to replay on duplicate request
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  expires_at    TIMESTAMPTZ  NOT NULL          -- created_at + INTERVAL '30 days'
);

CREATE INDEX idx_idempotency_keys_expires ON idempotency_keys (expires_at);
```

Purge job (runs daily):
```sql
DELETE FROM idempotency_keys WHERE expires_at < NOW();
```

---

## 6. `stripe_processed_events` Table

Webhook deduplication store. See `webhook-handler.md` for the full ingestion flow.

```sql
CREATE TABLE stripe_processed_events (
  event_id    VARCHAR(64)  PRIMARY KEY,       -- evt_xxx from Stripe
  received_at TIMESTAMPTZ  NOT NULL,
  event_type  VARCHAR(128) NOT NULL
);

CREATE INDEX idx_stripe_events_received_at ON stripe_processed_events (received_at);
```

Purge job (runs daily):
```sql
DELETE FROM stripe_processed_events WHERE received_at < NOW() - INTERVAL '30 days';
```

---

## 7. Atomicity Requirements

The following operations must be executed within a single database transaction:

| Operation | Entities Modified Atomically |
|---|---|
| `initiatePayment` | `orders.status`, `orders.attempt_count`, `orders.version`, new `payment_attempts` row, new `idempotency_keys` row |
| `onPaymentSucceeded` (webhook) | `payment_attempts.status`, `payment_attempts.stripe_charge_id`, `orders.status`, `orders.version`, `stripe_processed_events` insert |
| `onPaymentFailed` (webhook) | `payment_attempts.status`, `orders.attempt_count` (if applicable), `stripe_processed_events` insert |
| `cancelOrder` | `orders.status`, `orders.version`, `inventory.release` event publish (transactional outbox pattern recommended) |
| `initiateRefund` | `refund_records` insert, `idempotency_keys` insert |
| `onChargeRefunded` (webhook) | `refund_records.status`, `payment_attempts.status`, `orders.status`, `stripe_processed_events` insert |

### Transactional Outbox Pattern

For operations that must emit domain events (e.g., `inventory.release`, `order.confirmed`) alongside database changes, use the transactional outbox pattern:

1. Write the domain event to an `outbox_events` table **within the same transaction** as the state change.
2. A background relay process reads undelivered outbox events and publishes them to the event bus.
3. This guarantees that the event is emitted if and only if the database transaction commits.

---

## 8. Retention and Purge Schedule

| Table | Retention | Purge Condition |
|---|---|---|
| `orders` | 7 years (regulatory) | Manual or archival pipeline |
| `payment_attempts` | 13 months | `created_at < NOW() - INTERVAL '13 months'` |
| `refund_records` | 13 months | `created_at < NOW() - INTERVAL '13 months'` |
| `idempotency_keys` | 30 days | `expires_at < NOW()` |
| `stripe_processed_events` | 30 days | `received_at < NOW() - INTERVAL '30 days'` |

All purge jobs run daily as scheduled background tasks. Retention periods align with `data-classification.md`.
