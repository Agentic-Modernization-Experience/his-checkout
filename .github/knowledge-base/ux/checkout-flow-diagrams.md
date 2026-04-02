# Checkout Flow Diagrams

**Phase 1 Deliverable — Checkout UX & Flow Design**
**Profile**: Hosted Checkout | **Payment Methods**: credit_card, pix, boleto | **Provider**: Stripe | **Currency**: BRL

---

## 1. Primary Checkout Flow — Page and Modal Sequence

```
[Cart Page]
     │
     ▼
[Checkout Entry]
     │
     ├──────────────────────┐
     ▼                      ▼
[Guest Flow]        [Returning User Flow]
     │                      │
     ▼                      ▼
[Contact Info]      [Saved Methods List]
     │                      │
     ▼                      │
[Delivery/Address]          │
     │                      │
     └──────────┬───────────┘
                ▼
        [Payment Method Selection]
                │
     ┌──────────┼──────────┐
     ▼          ▼          ▼
[Credit Card] [Pix]   [Boleto]
     │          │          │
     ▼          ▼          ▼
[Card Form] [Pix QR]  [Boleto PDF]
     │       [Timer]   [Expiry]
     │          │          │
     └──────────┴──────────┘
                ▼
        [Order Review & Confirm]
                │
                ▼
        [Submit / Place Order]
                │
     ┌──────────┴──────────┐
     ▼                      ▼
[Success / Confirmation]  [Error / Recovery]
```

---

## 2. Guest vs. Returning User Branching

### 2.1 Guest Checkout Path

| Step | Screen / Component | Required Fields | Notes |
|------|-------------------|-----------------|-------|
| 1 | Contact Information | Email, Full Name, Phone | Email used for order confirmation |
| 2 | Delivery / Billing Address | Street, Number, Complement, City, State, Postal Code | Autocomplete via CEP (Brazil) |
| 3 | Payment Method Selection | — | Shows credit_card, pix, boleto |
| 4 | Payment Details | Varies by method | See section 3 |
| 5 | Order Review | — | Read-only summary; edit links per section |
| 6 | Confirmation | — | Order number, estimated time, next steps |

### 2.2 Returning User Path

| Step | Screen / Component | Notes |
|------|-------------------|-------|
| 1 | Authenticated landing | Skip contact step if profile is complete |
| 2 | Address selection | Pre-fill last-used address; option to add new |
| 3 | Saved Methods List | Shows masked card(s); option to use new method |
| 4 | Payment confirmation | One-tap confirm for saved card with CVV re-entry |
| 5 | Order Review | Same as guest |
| 6 | Confirmation | Same as guest |

**Branch decision point**: Presence of a valid session token. If session expires mid-checkout, surface inline session-expired notice and offer re-authentication without losing cart state.

---

## 3. Payment Method Step Transitions

### 3.1 Credit Card Path

```
[Method Selection: Credit Card]
          │
          ▼
[Card Form]
  - Cardholder Name
  - Card Number (Stripe Elements / tokenized)
  - Expiry MM/YY
  - CVV
  - Installments selector (if enabled)
          │
          ▼
[Tokenization via Stripe.js]
  → stripe.createToken() or confirmCardPayment()
          │
    ┌─────┴──────┐
    ▼            ▼
[Token OK]  [Tokenization Error]
    │            │
    ▼            ▼
[Submit]    [Inline field error — user corrects]
    │
    ▼
[Backend authorization]
    │
  ┌─┴─────────┬───────────┐
  ▼           ▼            ▼
[Authorized] [Declined]  [Network Error]
  │           │            │
  ▼           ▼            ▼
[Confirm]  [Error msg + [Retry modal]
           retry option]
```

### 3.2 Pix Path

```
[Method Selection: Pix]
          │
          ▼
[Order Review shown]
          │
          ▼
[Submit → Backend creates Pix charge via Stripe]
          │
          ▼
[Pix QR Code Modal]
  - QR code image
  - Copy-paste Pix code button
  - Countdown timer (default: 30 minutes)
  - "I've paid" button (optional polling trigger)
          │
    ┌─────┴──────┐
    ▼            ▼
[Payment       [Timer
Confirmed]     Expired]
    │            │
    ▼            ▼
[Confirm]   [Regenerate Pix
             option + new timer]
```

### 3.3 Boleto Path

```
[Method Selection: Boleto]
          │
          ▼
[CPF/CNPJ entry] (if not already captured)
          │
          ▼
[Submit → Backend creates Boleto via Stripe]
          │
          ▼
[Boleto Issued Screen]
  - Barcode display
  - "Download PDF" button
  - "Copy barcode" button
  - Expiry date (typically 1–3 business days)
  - Email confirmation note
          │
          ▼
[Confirmation Page]
  (Order status: "Awaiting payment")
```

---

## 4. Recovery and Fallback Paths

### 4.1 Declined Card Recovery

```
[Declined Response from Stripe]
          │
          ▼
[Error Presented Inline]
  - User-facing message (see UX Copy Guidance)
  - No page reload
          │
    ┌─────┴──────────────┐
    ▼                    ▼
[Try Different Card] [Try Different Method]
    │                    │
    ▼                    ▼
[Card Form (cleared)] [Method Selection]
```

**Idempotency rule**: Backend must use an idempotency key per order+attempt combination. Re-submitting the same card form must not create a duplicate order.

### 4.2 Network Failure Recovery

```
[Request Timeout / 5xx]
          │
          ▼
[Retry Modal]
  - "Something went wrong" message
  - [Retry] button (re-submits same payload with same idempotency key)
  - [Cancel] button (returns to cart)
          │
    ┌─────┴──────┐
    ▼            ▼
[Retry OK]  [Retry fails again]
    │            │
    ▼            ▼
[Continue]  [Support contact option]
```

### 4.3 Session Resume / Interrupted Checkout

- Cart state is persisted in local storage and server-side session for 24 hours.
- Returning to checkout URL within the window restores the user to the last completed step.
- Payment fields are **never** pre-filled on resume (security requirement).
- If order was already submitted and pending, redirect to confirmation page instead of re-presenting form.

---

## 5. Modal Inventory

| Modal | Trigger | Dismissible | Notes |
|-------|---------|-------------|-------|
| Pix QR Code | Pix submission success | Yes (with confirmation) | Dismissing warns order still pending |
| Retry / Network Error | 5xx / timeout | Yes | Returns to last form state |
| Session Expired | Auth timeout mid-flow | No (must re-auth) | Cart state preserved |
| Unsaved Changes | Navigating away mid-form | Yes | Browser `beforeunload` + custom modal |
| Order Already Submitted | Duplicate submission detected | No | Redirects to confirmation |
