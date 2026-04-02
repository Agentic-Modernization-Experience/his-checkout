# Error Handling Strategy — Client-Safe Failure Mapping

**Phase 5 Deliverable — Frontend Checkout Implementation**
**Provider**: Stripe | **Profile**: Hosted Checkout | **Currency**: BRL

---

## 1. Design Principles

- **Never expose provider-specific error codes or messages to the user.** Raw Stripe error codes (e.g., `card_declined`, `insufficient_funds`) must be mapped to user-facing strings on the backend before reaching the frontend.
- **Never expose internal server errors in user-facing messages.** A 500 response is always presented as a generic "something went wrong" state.
- **Every error state must have a recoverable path.** The user must always be able to retry, correct their input, or return to the cart.
- **Submission state must be reset on error.** `isSubmitting` must return to `false` so the user is not permanently locked out of retrying.

---

## 2. Error Classification

### 2.1 Field-Level Errors (Client-Side Validation)

Detected before any network request is made. The form is not submitted.

| Source | Example | Presentation |
|--------|---------|-------------|
| Required field empty | Name not entered | Inline error below field; focus moves to first invalid field on submit |
| Format invalid | Invalid CEP, invalid CPF check digit | Inline error below field |
| Stripe Elements field error | Invalid card number (Luhn fail) | Inline error from `element.on('change', event.error.message)` |

Recovery: User corrects the field. Error clears on blur (or immediately on input for fields with a format rule).

---

### 2.2 Tokenization Errors (Stripe.js)

Returned synchronously from `stripe.confirmCardPayment()` before any backend call.

| `error.type` | `error.code` (examples) | Mapped User Message | Action |
|-------------|------------------------|--------------------|----|
| `card_error` | `incorrect_number` | "The card number entered is incorrect. Please check and try again." | Re-enable card form; clear number field |
| `card_error` | `invalid_expiry_year` | "The card's expiry date is not valid." | Re-enable form; focus expiry field |
| `card_error` | `invalid_cvc` | "The security code (CVV) is not valid." | Re-enable form; focus CVC field |
| `validation_error` | (any) | "Please check your card details and try again." | Re-enable form |
| `api_connection_error` | — | "We could not connect to our payment provider. Please try again." | Show retry modal |
| `api_error` | — | "An unexpected error occurred. Please try again." | Show retry modal |

- `error.message` from Stripe is used only for logging (server-side); a mapped message is shown to the user.
- Do not forward raw `error.code` to analytics; use mapped `decline_category` (see `analytics-implementation.md`).

---

### 2.3 Decline Errors (Authorization Failure)

The `PaymentIntent` reaches `requires_payment_method` status (authorization declined by the card network or issuer).

Backend maps Stripe's `decline_code` to a `decline_category` before sending it to the frontend:

| `decline_category` (from backend) | User-Facing Message | Recovery Path |
|----------------------------------|--------------------|----|
| `insufficient_funds` | "Payment was declined due to insufficient funds. Please try a different card or payment method." | Re-enable payment step; offer method change |
| `expired_card` | "This card has expired. Please use a different card." | Re-enable payment step |
| `incorrect_cvc` | "The security code is incorrect. Please re-enter your CVV." | Re-enable CVC field only |
| `lost_or_stolen` | "This card cannot be used. Please use a different payment method." | Re-enable payment step; no card retry |
| `do_not_honor` | "Your bank did not approve this payment. Please contact your bank or try a different method." | Re-enable payment step |
| `generic` | "Your payment was not approved. Please try a different payment method." | Re-enable payment step |

- The frontend only consumes `decline_category`; raw Stripe `decline_code` stays server-side.
- After a decline, `attemptNumber` increments; idempotency key becomes `{sessionId}-attempt-{n}` for the next try.

---

### 2.4 Backend Validation Errors (4xx)

| HTTP Status | `error_code` (from backend) | User-Facing Message | Action |
|------------|----------------------------|--------------------|----|
| 400 | `invalid_request` | "Some information in your order is invalid. Please review and try again." | Return user to review step |
| 409 | `order_already_exists` | "This order was already placed." | Redirect to confirmation page |
| 422 | `payment_method_not_supported` | "This payment method is not available for your order. Please choose another." | Return to payment step |
| 422 | `amount_mismatch` | "Your order total has changed. Please review your order before continuing." | Return to review step with updated total |

- 409 with `order_already_exists` must redirect to the confirmation page rather than showing an error. This handles duplicate submission from the user or from the network (retry of a successful request).

---

### 2.5 System Errors (5xx / Network Timeout)

Displayed as a retry modal, not an inline field error.

| Trigger | User-Facing Message | Retry Behavior |
|---------|--------------------|----|
| Network timeout (> 10s with no response) | "Something went wrong. Your payment may not have been processed." | Retry with same idempotency key (safe re-send) |
| 500 / 503 from backend | "We encountered a technical issue. Please try again." | Retry with same idempotency key |
| Backend unreachable | "We're having trouble reaching our servers. Please check your connection and try again." | Retry or return to cart |

**Retry modal UI**:
- "Try again" button: re-submits with the same idempotency key (prevents duplicate order).
- "Return to cart" button: clears session and navigates to cart page.
- After 3 consecutive system errors, show "Persistent issue" message with a support contact link.

---

## 3. Idempotency Key Behavior on Error

| Scenario | Key Behavior |
|----------|-------------|
| Network timeout (no response) | Reuse same key on retry — the original request may have succeeded |
| 5xx response (confirmed failed) | Reuse same key on retry — backend must be idempotent for these keys |
| Decline (clear failure) | New attempt key `{sessionId}-attempt-{n}` — the previous payment was definitively declined |
| User changes payment method | New attempt key — different method is a new payment attempt |
| 409 `order_already_exists` | Do not retry — redirect to confirmation |

---

## 4. Global Error State Flow

```
User clicks Submit
      │
      ▼
[isSubmitting = true]
[All inputs disabled]
      │
      ▼
[API call in flight]
      │
   ┌──┴───────────────────────────┐
   ▼                              ▼
[Success]                     [Failure]
   │                              │
[isSubmitting = false]         [isSubmitting = false]
[→ ConfirmationStep]           │
                            ┌───┴────────────────┐
                            ▼                    ▼
                    [Field/Decline error]  [System error]
                    [Inline message]       [Retry modal]
                    [Re-enable form]       [Retry or Cancel]
```

---

## 5. Error Message Copy Rules

- Messages are written in plain language, in Portuguese (pt-BR) for the Brazilian market.
- Messages never mention "Stripe", "payment gateway", or internal error codes.
- Messages always tell the user what happened and what they can do next.
- Messages do not use alarming language ("error", "failure" where possible) — prefer "could not", "wasn't approved", "please try again".

---

## 6. Refresh/Reload Recovery

When the user reloads the page during an in-progress checkout:

1. If `checkoutSessionId` exists in `sessionStorage` and no `orderResult` is stored: restore to last completed step.
2. If the reload happened during an in-flight request (browser closed during submission), the outcome is unknown.
3. On restore, `CheckoutSessionGuard` calls `GET /api/checkout/session/{sessionId}/status`:
   - `pending`: restore to last step; `isSubmitting = false`.
   - `confirmed`: redirect to `ConfirmationStep` (order succeeded).
   - `failed`: restore to payment step with a soft error message: "Your previous payment attempt did not complete. Please try again."
   - `not_found`: treat as a fresh checkout start.

This prevents the user from unknowingly placing a duplicate order after a network interruption.
