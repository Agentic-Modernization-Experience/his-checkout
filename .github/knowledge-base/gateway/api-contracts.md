# API Contract Specification — Checkout & Payment Gateway

**Phase**: 3 — Gateway & API Contracts
**Checkout Mode**: Hosted
**Provider**: Stripe
**Currency**: BRL
**Payment Methods**: credit_card, pix, boleto

---

## 1. Overview

This document defines the stable, versioned API surface between the first-party checkout frontend, the first-party backend, and the Stripe payment provider. All contracts are designed to be backward-compatible, idempotent-safe, and provider-independent from the consumer's perspective.

---

## 2. Base URL and Versioning

```
Base URL:  /api/v1
Content-Type: application/json
Accept: application/json
```

All breaking changes increment the version prefix (`v2`, `v3`, …). Non-breaking additions (new optional fields, new response fields) are made within the existing version.

---

## 3. Checkout Session Lifecycle Endpoints

### 3.1 Create Checkout Session

Initialises a new checkout session for the order. Must be called before any payment step.

```
POST /api/v1/checkout/sessions
```

#### Request

| Field | Type | Required | Description |
|---|---|---|---|
| `order_id` | string | Yes | Merchant-generated unique order identifier |
| `currency` | string | Yes | ISO 4217 currency code (`BRL`) |
| `amount` | integer | Yes | Total amount in smallest currency unit (centavos) |
| `customer` | object | Yes | Customer identity block (see below) |
| `items` | array | Yes | Line items; at least one required |
| `metadata` | object | No | Arbitrary key-value pairs (max 20 keys, 500 chars/value) |

**`customer` object**

| Field | Type | Required | Description |
|---|---|---|---|
| `email` | string | Yes | Contact email; used for order confirmation |
| `name` | string | Yes | Full name |
| `phone` | string | No | E.164 format |
| `tax_id` | string | Conditional | CPF/CNPJ — required for boleto |

**`items` array element**

| Field | Type | Required | Description |
|---|---|---|---|
| `sku` | string | Yes | Product identifier |
| `description` | string | Yes | Display name |
| `quantity` | integer | Yes | Positive integer |
| `unit_amount` | integer | Yes | Unit price in centavos |

**Example Request**

```json
{
  "order_id": "ord_abc123",
  "currency": "BRL",
  "amount": 29900,
  "customer": {
    "email": "user@example.com",
    "name": "João Silva",
    "phone": "+5511999990000",
    "tax_id": "123.456.789-09"
  },
  "items": [
    {
      "sku": "PROD-001",
      "description": "Produto Exemplo",
      "quantity": 1,
      "unit_amount": 29900
    }
  ]
}
```

#### Response — 201 Created

| Field | Type | Description |
|---|---|---|
| `session_id` | string | Opaque session identifier (`cs_xxx`) |
| `order_id` | string | Echo of the request `order_id` |
| `status` | string | Always `pending` on creation |
| `expires_at` | string | ISO 8601 UTC — session expiry (30 minutes from creation) |
| `client_secret` | string | Stripe PaymentIntent client secret for Stripe.js (hosted mode) |
| `payment_methods` | array | Enabled methods for this session: `credit_card`, `pix`, `boleto` |

**Example Response**

```json
{
  "session_id": "cs_live_abc123",
  "order_id": "ord_abc123",
  "status": "pending",
  "expires_at": "2024-01-15T14:30:00Z",
  "client_secret": "pi_3OxYz_secret_abc123",
  "payment_methods": ["credit_card", "pix", "boleto"]
}
```

---

### 3.2 Retrieve Checkout Session

Returns the current state of an existing checkout session.

```
GET /api/v1/checkout/sessions/{session_id}
```

#### Response — 200 OK

| Field | Type | Description |
|---|---|---|
| `session_id` | string | Session identifier |
| `order_id` | string | Associated order identifier |
| `status` | string | Current session status (see §4 lifecycle) |
| `payment_intent_id` | string | Stripe PaymentIntent ID (`pi_xxx`); present once initiated |
| `payment_method` | string | Chosen payment method; present once selected |
| `expires_at` | string | ISO 8601 UTC |
| `created_at` | string | ISO 8601 UTC |
| `updated_at` | string | ISO 8601 UTC |

---

### 3.3 Expire Checkout Session (Cancel)

Explicitly cancels an open session. Idempotent — safe to call if already expired.

```
DELETE /api/v1/checkout/sessions/{session_id}
```

#### Response — 200 OK

```json
{ "session_id": "cs_live_abc123", "status": "expired" }
```

---

## 4. Payment Intent Lifecycle Endpoints

### 4.1 Confirm Payment Intent

Called after Stripe.js tokenises the payment method and the frontend has a confirmed `paymentMethod` or `paymentIntent` result.

```
POST /api/v1/payments/intents/{payment_intent_id}/confirm
```

#### Request Headers (Required)

| Header | Description |
|---|---|
| `Idempotency-Key` | See §7 — Idempotency Key Behavior |

#### Request

| Field | Type | Required | Description |
|---|---|---|---|
| `session_id` | string | Yes | Checkout session the payment belongs to |
| `payment_method_id` | string | Conditional | Stripe `pm_xxx` — required for credit_card |
| `payment_method_type` | string | Yes | `credit_card`, `pix`, or `boleto` |

**Example Request (credit card)**

```json
{
  "session_id": "cs_live_abc123",
  "payment_method_id": "pm_1OxYz_abc123",
  "payment_method_type": "credit_card"
}
```

#### Response — 202 Accepted

| Field | Type | Description |
|---|---|---|
| `payment_intent_id` | string | Stripe PaymentIntent ID |
| `order_id` | string | Associated order |
| `status` | string | Internal payment status (see §4.3) |
| `next_action` | object | Present when further client action is required (3DS, Pix QR, boleto details) |

**`next_action` for Pix**

```json
{
  "type": "pix",
  "pix": {
    "qr_code": "00020126360014...",
    "qr_code_image_url": "https://...",
    "expires_at": "2024-01-15T14:30:00Z"
  }
}
```

**`next_action` for Boleto**

```json
{
  "type": "boleto",
  "boleto": {
    "barcode": "23793.38128 60007.827136 95000.063305 1 99630000029900",
    "pdf_url": "https://...",
    "expires_at": "2024-01-18T23:59:59Z"
  }
}
```

**`next_action` for 3DS (credit card redirect)**

```json
{
  "type": "redirect_to_url",
  "redirect_to_url": {
    "url": "https://hooks.stripe.com/...",
    "return_url": "https://checkout.example.com/return"
  }
}
```

---

### 4.2 Retrieve Payment Status

Retrieves the latest payment status for an order. Used for polling (Pix / boleto) and for post-redirect status verification.

```
GET /api/v1/payments/intents/{payment_intent_id}
```

#### Response — 200 OK

| Field | Type | Description |
|---|---|---|
| `payment_intent_id` | string | Stripe PaymentIntent ID |
| `order_id` | string | Associated order |
| `status` | string | Internal status (see §4.3) |
| `amount` | integer | Amount in centavos |
| `currency` | string | ISO 4217 code |
| `payment_method_type` | string | Method used |
| `created_at` | string | ISO 8601 UTC |
| `updated_at` | string | ISO 8601 UTC |

---

### 4.3 Canonical Status Lifecycle

The following internal statuses are used across all API responses. They map from Stripe's PaymentIntent statuses and must not expose raw Stripe statuses to API consumers.

| Internal Status | Description | Terminal |
|---|---|---|
| `pending` | Payment not yet initiated | No |
| `processing` | Stripe is processing the payment | No |
| `requires_action` | Client action required (3DS, Pix payment) | No |
| `requires_capture` | Authorised; awaiting manual capture | No |
| `confirmed` | Payment successfully captured | Yes |
| `failed` | Payment rejected or declined | Yes |
| `cancelled` | Order or payment was cancelled | Yes |
| `refunded` | Charge fully refunded | Yes |
| `partially_refunded` | Charge partially refunded | Yes |

**Stripe → Internal Status Mapping**

| Stripe `PaymentIntent.status` | Internal Status | Notes |
|---|---|---|
| `requires_payment_method` | `pending` | Before method attached |
| `requires_confirmation` | `pending` | After method attached, before confirm |
| `requires_action` | `requires_action` | 3DS or async payment pending |
| `processing` | `processing` | Backend processing in progress |
| `requires_capture` | `requires_capture` | Manual-capture flow only |
| `succeeded` | `confirmed` | Primary success path |
| `canceled` | `cancelled` | Via Stripe or backend cancellation |

**Stripe Charge/Webhook → Internal Transition**

| Stripe Event | From State | To State |
|---|---|---|
| `payment_intent.succeeded` | `processing` | `confirmed` |
| `payment_intent.payment_failed` | `processing` / `requires_action` | `failed` |
| `payment_intent.requires_action` | `processing` | `requires_action` |
| `payment_intent.canceled` | any non-terminal | `cancelled` |
| `charge.refunded` (full) | `confirmed` | `refunded` |
| `charge.refunded` (partial) | `confirmed` | `partially_refunded` |

Transitions must be guarded: a transition is only applied if the current status is in the valid `From State`. Out-of-order or duplicate events are silently ignored.

---

## 5. Order Status Endpoint

### 5.1 Retrieve Order

```
GET /api/v1/orders/{order_id}
```

#### Response — 200 OK

| Field | Type | Description |
|---|---|---|
| `order_id` | string | Order identifier |
| `status` | string | Order status (see below) |
| `payment_status` | string | Internal payment status (§4.3) |
| `payment_intent_id` | string | Stripe PaymentIntent ID |
| `amount` | integer | Centavos |
| `currency` | string | ISO 4217 |
| `items` | array | Ordered items (same shape as creation request) |
| `customer` | object | Customer block (email, name) |
| `created_at` | string | ISO 8601 UTC |
| `updated_at` | string | ISO 8601 UTC |

**Order Status Values**

| Order Status | Description |
|---|---|
| `created` | Order record exists; payment not yet initiated |
| `pending_payment` | Payment initiated; awaiting confirmation |
| `confirmed` | Payment captured; order ready for fulfillment |
| `cancelled` | Order and/or payment cancelled |
| `refunded` | Order fully refunded |

---

## 6. Webhook Receiver Endpoint

All Stripe webhook events are delivered to this endpoint. The backend must verify the Stripe signature before processing (see `webhook-verification.md`).

```
POST /api/v1/webhooks/stripe
Content-Type: application/json
Stripe-Signature: <hmac>
```

#### Handled Events

| Stripe Event | Internal Action |
|---|---|
| `payment_intent.succeeded` | Transition order → `confirmed`; payment → `confirmed` |
| `payment_intent.payment_failed` | Payment → `failed`; trigger recovery notification |
| `payment_intent.requires_action` | Payment → `requires_action`; notify client |
| `payment_intent.canceled` | Payment → `cancelled`; order → `cancelled` |
| `charge.refunded` | Payment → `refunded` or `partially_refunded` |
| `charge.dispute.created` | Flag order for dispute review |

#### Responses

| Scenario | HTTP Code |
|---|---|
| Signature verification failed | 400 |
| Duplicate event (already processed) | 200 |
| Event type not handled | 200 |
| Event processed successfully | 200 |
| Transient processing error | 500 (Stripe retries) |
