# Error Envelope Definition — Checkout API

**Phase**: 3 — Gateway & API Contracts
**Provider**: Stripe
**Applies to**: All `/api/v1/` endpoints

---

## 1. Error Envelope Structure

All API errors are returned as a consistent JSON envelope. Consumers must never parse HTTP status codes alone — the `error` object is always present and authoritative for error handling.

```json
{
  "error": {
    "code": "payment_declined",
    "message": "Your payment was declined. Please try a different payment method.",
    "request_id": "req_abc123xyz",
    "details": []
  }
}
```

| Field | Type | Always Present | Description |
|---|---|---|---|
| `error.code` | string | Yes | Machine-readable error code (see §2) |
| `error.message` | string | Yes | Human-readable, client-safe message in the request locale |
| `error.request_id` | string | Yes | Correlation ID for support tracing — matches `X-Request-ID` response header |
| `error.details` | array | Yes | Field-level validation errors; empty array when not applicable |

### `details` Array Element

| Field | Type | Description |
|---|---|---|
| `field` | string | Dot-notation path to the offending field (e.g., `customer.tax_id`) |
| `code` | string | Field-level error code (e.g., `required`, `invalid_format`) |
| `message` | string | Human-readable field error |

**Example — Validation Error**

```json
{
  "error": {
    "code": "validation_error",
    "message": "The request contains invalid fields.",
    "request_id": "req_abc123xyz",
    "details": [
      {
        "field": "customer.tax_id",
        "code": "invalid_format",
        "message": "CPF/CNPJ must be a valid Brazilian tax identifier."
      },
      {
        "field": "amount",
        "code": "must_be_positive",
        "message": "Amount must be greater than zero."
      }
    ]
  }
}
```

---

## 2. Error Code Registry

### 2.1 Client Errors (4xx)

| Code | HTTP Status | Description | Retryable |
|---|---|---|---|
| `validation_error` | 400 | One or more request fields are invalid (see `details`) | No |
| `invalid_idempotency_key` | 400 | Idempotency key format is invalid | No |
| `idempotency_conflict` | 409 | Same idempotency key used with a different request payload | No |
| `session_not_found` | 404 | Checkout session does not exist | No |
| `session_expired` | 410 | Checkout session has expired | No |
| `order_not_found` | 404 | Order does not exist | No |
| `payment_intent_not_found` | 404 | PaymentIntent does not exist | No |
| `payment_method_required` | 400 | `payment_method_id` is required for credit card payments | No |
| `payment_declined` | 402 | Stripe declined the payment | No (user action) |
| `payment_cancelled` | 409 | PaymentIntent has already been cancelled | No |
| `payment_already_confirmed` | 409 | PaymentIntent has already been confirmed | No |
| `insufficient_funds` | 402 | Card declined due to insufficient funds | No (user action) |
| `card_expired` | 402 | Card is past its expiry date | No (user action) |
| `do_not_honor` | 402 | Generic card decline — bank declined without specific reason | No (user action) |
| `authentication_required` | 402 | 3DS authentication is required and was not completed | No (user action) |
| `fraud_blocked` | 402 | Payment blocked by fraud detection rules | No |
| `duplicate_order` | 409 | An order with this `order_id` already exists | No |
| `currency_not_supported` | 400 | Currency is not supported for the selected payment method | No |
| `amount_too_small` | 400 | Amount is below the minimum for the payment method | No |
| `amount_too_large` | 400 | Amount exceeds the maximum for the payment method | No |
| `unauthorized` | 401 | Missing or invalid authentication credentials | No |
| `forbidden` | 403 | Authenticated but lacks permission for this operation | No |
| `rate_limited` | 429 | Too many requests — see `Retry-After` header | Yes (after delay) |

### 2.2 Server Errors (5xx)

| Code | HTTP Status | Description | Retryable |
|---|---|---|---|
| `internal_error` | 500 | Unexpected server error | Yes |
| `gateway_error` | 502 | Stripe API returned an unexpected error | Yes |
| `gateway_timeout` | 504 | Stripe API did not respond within the timeout window | Yes |
| `service_unavailable` | 503 | Service temporarily unavailable | Yes (after delay) |

### 2.3 Stripe-Specific Decline Code Mapping

When Stripe declines a charge, the internal `error.code` is mapped from Stripe's `decline_code` as follows:

| Stripe `decline_code` | Internal `error.code` |
|---|---|
| `insufficient_funds` | `insufficient_funds` |
| `expired_card` | `card_expired` |
| `card_declined` (generic) | `payment_declined` |
| `do_not_honor` | `do_not_honor` |
| `authentication_required` | `authentication_required` |
| `stolen_card` | `fraud_blocked` |
| `lost_card` | `fraud_blocked` |
| `pickup_card` | `fraud_blocked` |
| `fraudulent` | `fraud_blocked` |
| *(all others)* | `payment_declined` |

Stripe's raw `decline_code` must **never** be forwarded to the client response. Only the mapped internal code is returned.

---

## 3. HTTP Status Code Usage

| Status | When Used |
|---|---|
| 200 | Successful GET, DELETE |
| 201 | Successful resource creation (POST) |
| 202 | Accepted — async processing initiated (confirm payment) |
| 400 | Request validation failure |
| 401 | Missing or invalid authentication |
| 402 | Payment failure (decline, insufficient funds) |
| 403 | Authorization failure |
| 404 | Resource not found |
| 409 | Conflict (idempotency conflict, duplicate, wrong state) |
| 410 | Resource gone (expired session) |
| 429 | Rate limited |
| 500 | Internal server error |
| 502 | Bad gateway (upstream Stripe error) |
| 503 | Service temporarily unavailable |
| 504 | Gateway timeout (upstream Stripe timeout) |

---

## 4. Client-Safe Message Policy

All `error.message` strings returned to clients must follow these rules:

1. **No internal detail** — stack traces, SQL errors, internal service names, or Stripe raw error messages must never appear in client-facing messages.
2. **Actionable** — when the user can resolve the issue, the message guides them (e.g., "Try a different payment method.").
3. **Non-technical** — messages are written in user-facing language, not developer language.
4. **Localised** — messages should honour the `Accept-Language` request header when internationalisation is supported.
5. **Stable** — message strings should not be used for programmatic logic; use `error.code` for that.

### Reference Messages by Code

| Code | Reference Client Message (pt-BR) |
|---|---|
| `payment_declined` | "Seu pagamento foi recusado. Verifique os dados ou tente outro método." |
| `insufficient_funds` | "Saldo insuficiente. Tente outro cartão ou método de pagamento." |
| `card_expired` | "Cartão vencido. Verifique a data de validade ou use outro cartão." |
| `authentication_required` | "Autenticação necessária. Siga as instruções do seu banco." |
| `fraud_blocked` | "Não foi possível processar o pagamento. Entre em contato com seu banco." |
| `session_expired` | "Sua sessão de pagamento expirou. Inicie o checkout novamente." |
| `rate_limited` | "Muitas tentativas. Aguarde um momento e tente novamente." |
| `internal_error` | "Ocorreu um erro inesperado. Tente novamente em alguns instantes." |
| `gateway_timeout` | "O serviço de pagamento está demorando para responder. Tente novamente." |

---

## 5. Response Headers

Every API response includes the following headers:

| Header | Description |
|---|---|
| `X-Request-ID` | UUID v4 — echoes the `request_id` in the error envelope |
| `X-Trace-ID` | Distributed trace identifier for backend correlation |
| `Retry-After` | Seconds to wait before retrying (present on 429 and 503 only) |
| `X-Idempotency-Replayed` | `true` if the response is a cached replay of a prior idempotent request |

---

## 6. Non-Error Response Contract

Successful responses must **never** include an `error` key. The absence of the `error` key at the response root is the authoritative signal of success.

```json
// SUCCESS — no "error" key
{
  "session_id": "cs_live_abc123",
  "order_id": "ord_abc123",
  "status": "pending"
}

// ERROR — "error" key always present
{
  "error": {
    "code": "session_not_found",
    "message": "Sessão de pagamento não encontrada.",
    "request_id": "req_xyz789",
    "details": []
  }
}
```
