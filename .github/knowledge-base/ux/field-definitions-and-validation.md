# Field Definitions & Validation Rules

**Phase 1 Deliverable — Checkout UX & Flow Design**
**Profile**: Hosted Checkout | **Payment Methods**: credit_card, pix, boleto | **Provider**: Stripe | **Currency**: BRL

---

## 1. Contact Information Fields (Guest Only)

| Field | Type | Required | Max Length | Validation Rule | Inline Error |
|-------|------|----------|------------|-----------------|--------------|
| Full Name | Text | Yes | 100 | Min 2 chars; letters, spaces, hyphens, apostrophes only | "Please enter your full name" |
| Email | Email | Yes | 254 | RFC 5322 format; real-time format check on blur | "Please enter a valid email address" |
| Phone | Tel | Yes | 20 | Brazilian format: (XX) XXXXX-XXXX; digits only after masking | "Please enter a valid phone number" |

**Validation timing**: Validate on `blur` (field loses focus). Re-validate on submit. Do not validate on keypress.

---

## 2. Delivery / Billing Address Fields

| Field | Type | Required | Max Length | Validation Rule | Inline Error |
|-------|------|----------|------------|-----------------|--------------|
| Postal Code (CEP) | Text | Yes | 9 (with hyphen) | Format: XXXXX-XXX; trigger address autocomplete on valid CEP | "Please enter a valid CEP" |
| Street | Text | Yes | 200 | Min 3 chars | "Please enter the street name" |
| Number | Text | Yes | 10 | Alphanumeric; "S/N" accepted for unnumbered | "Please enter the address number" |
| Complement | Text | No | 100 | Free text | — |
| Neighborhood | Text | Yes | 100 | Auto-filled from CEP lookup; editable | "Please enter the neighborhood" |
| City | Text | Yes | 100 | Auto-filled from CEP lookup; read-only if auto-filled | "Please enter the city" |
| State | Select | Yes | 2 (UF) | Brazilian state abbreviations (AC–TO) | "Please select a state" |

**CEP Autocomplete Behavior**:
- On valid 8-digit CEP entry: call ViaCEP (or equivalent) API.
- Auto-fill Street, Neighborhood, City, State if API returns data.
- If API fails: allow manual entry of all fields; show no error (silent fallback).
- If CEP not found: show "CEP not found — please fill in address manually" below the field.

---

## 3. Credit Card Fields

All card input fields use **Stripe Elements** (iframe-hosted). The following describes UX contract only; actual DOM elements are Stripe-managed.

| Field | Type | Stripe Element | Validation | Inline Error |
|-------|------|---------------|------------|--------------|
| Cardholder Name | Text (our DOM) | n/a | Min 2 chars; same rules as Full Name | "Please enter the name as shown on the card" |
| Card Number | Stripe Element | `CardNumberElement` | Stripe-side: Luhn + BIN check | Stripe-provided error message |
| Expiry Date | Stripe Element | `CardExpiryElement` | Stripe-side: not in past | Stripe-provided error message |
| CVV / CVC | Stripe Element | `CardCvcElement` | Stripe-side: 3–4 digits | Stripe-provided error message |
| Installments | Select (our DOM) | n/a | Valid option from server-provided list | "Please select a payment plan" |

**Tokenization contract**:
- Never read or log card values from Stripe Elements.
- Call `stripe.confirmCardPayment(clientSecret, { payment_method: { card: cardElement } })` on submit.
- On Stripe error, display `error.message` inline below the card number field.
- On success, forward `paymentIntent.id` to the backend for fulfillment.

**CVV Re-entry for Saved Cards**:
- Returning users selecting a saved card must re-enter CVV (3 digits) before confirming.
- CVV field uses `CardCvcElement` (Stripe-hosted).
- CVV is never stored or logged.

---

## 4. Pix Fields

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| (none) | — | — | Pix requires no payment data entry from the user. Charge is created server-side on submit. |

**Post-submit display (QR Modal)**:
- QR code image (base64 from Stripe Pix charge response).
- `pixCopyPasteCode` text with copy-to-clipboard button.
- Countdown timer (`expiresAt` from charge response, default 30 min).
- No field validation required.

---

## 5. Boleto Fields

| Field | Type | Required | Max Length | Validation Rule | Inline Error |
|-------|------|----------|------------|-----------------|--------------|
| CPF / CNPJ | Text | Yes | 18 (with mask) | CPF: XXX.XXX.XXX-XX; CNPJ: XX.XXX.XXX/XXXX-XX; validate check digit | "Please enter a valid CPF or CNPJ" |

**CPF/CNPJ Validation Logic**:
- Auto-detect CPF (11 digits) vs CNPJ (14 digits) based on length.
- Apply Brazilian check-digit algorithm (modulo 11 for CPF; standard CNPJ check for CNPJ).
- Validate on blur; error cleared on focus.

**Post-submit display**:
- Boleto barcode as text (copyable).
- "Download PDF" link (opens Stripe boleto PDF URL).
- Expiry date prominently displayed.
- No additional field validation after submission.

---

## 6. Submission States

| State | Trigger | UI Behavior |
|-------|---------|-------------|
| **Idle** | Default | All fields enabled; Submit button enabled if form is valid |
| **Validating** | Submit pressed | Client-side validation runs; invalid fields show inline errors; form not submitted |
| **Loading / Submitting** | All fields valid; request in flight | Submit button disabled + spinner; all fields disabled; no duplicate submission possible |
| **Success** | Backend confirms payment | Redirect to confirmation page |
| **Error (Field)** | Stripe tokenization error or field-level API error | Inline error shown; fields re-enabled; submit re-enabled |
| **Error (System)** | 5xx / network timeout | Retry modal shown; form state preserved |
| **Declined** | Payment authorization declined | Decline message shown inline; form re-enabled for retry |

---

## 7. Retry and Idempotency Rules

- Each checkout session is assigned an **idempotency key** on first load (UUID v4, stored in session).
- On retry after a network error, the **same idempotency key** is reused — this prevents duplicate orders.
- On user-initiated "try a different card" or "try a different method", a **new attempt token** is appended (e.g., `{sessionKey}-attempt-2`), still scoped to the same order.
- Backend must return `409 Conflict` if an order with the same key was already created; frontend redirects to confirmation.

---

## 8. Promo / Coupon Entry (if applicable)

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Promo Code | Text | No | Uppercase; alphanumeric + hyphens; max 30 chars |

**Behavior**:
- Validate on "Apply" button click (not on blur).
- Show inline success ("Coupon applied: 10% off") or error ("Invalid or expired code") below the field.
- Applied coupon reflected in Order Summary total before submission.
- Coupon code is passed to backend at order submission, not pre-validated on front end only.
