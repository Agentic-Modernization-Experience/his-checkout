# Component Architecture — Checkout Frontend

**Phase 5 Deliverable — Frontend Checkout Implementation**
**Profile**: Hosted Checkout | **Payment Methods**: credit_card, pix, boleto | **Provider**: Stripe | **Currency**: BRL

---

## 1. Component Tree Overview

```
<CheckoutApp>                        ← Root; owns global checkout state
  ├── <CheckoutSessionGuard>         ← Handles session recovery and duplicate-submission guard
  ├── <CheckoutProgress>             ← Step indicator (aria-label, step n of N)
  └── <CheckoutStepRouter>           ← Renders the active step; manages step transitions
        ├── <SummaryStep>            ← Order summary (read-only; shows items, subtotal, shipping)
        ├── <ContactStep>            ← Guest contact info (name, email, phone)
        ├── <AddressStep>            ← Delivery/billing address with CEP autocomplete
        ├── <PaymentStep>            ← Payment method selection + per-method sub-components
        │     ├── <PaymentMethodSelector>   ← Radio group: credit_card / pix / boleto
        │     ├── <CreditCardForm>          ← Stripe Elements card form
        │     ├── <PixSummaryConfirm>       ← Pix order review before submit
        │     └── <BoletoDataForm>          ← CPF/CNPJ entry for Boleto
        ├── <ReviewStep>             ← Read-only order review; edit links per section
        └── <ConfirmationStep>       ← Post-submit outcome; branches by payment method
              ├── <CreditCardConfirmation>  ← Authorized / 3DS redirect / declined states
              ├── <PixQrModal>              ← QR code, copy code, countdown timer
              └── <BoletoIssuedScreen>      ← Barcode, PDF download, expiry notice
```

---

## 2. State Ownership

### 2.1 Global Checkout State (`CheckoutApp`)

| State Slice | Type | Owner | Notes |
|-------------|------|-------|-------|
| `checkoutSessionId` | `string` | `CheckoutApp` | UUID v4; persisted to `sessionStorage` on init; used as idempotency key |
| `attemptNumber` | `number` | `CheckoutApp` | Increments on each user-initiated retry; appended to idempotency key |
| `activeStep` | `StepName` | `CheckoutApp` | Controls which step is rendered |
| `completedSteps` | `Set<StepName>` | `CheckoutApp` | Enables non-linear navigation back to completed steps |
| `isSubmitting` | `boolean` | `CheckoutApp` | Prevents duplicate submissions; disables all inputs during in-flight request |
| `orderResult` | `OrderResult \| null` | `CheckoutApp` | Set on backend confirmation; drives `ConfirmationStep` content |
| `globalError` | `ClientSafeError \| null` | `CheckoutApp` | System-level errors (5xx, network); shown in retry modal |

### 2.2 Step-Local State

Each step component owns its own form state and inline validation state. Step data is lifted to `CheckoutApp` only when the step is confirmed (user clicks "Continue").

| Component | Local State | Notes |
|-----------|------------|-------|
| `ContactStep` | `{ name, email, phone }` + field errors | Validated on blur; lifted on continue |
| `AddressStep` | `{ cep, street, number, complement, neighborhood, city, state }` + CEP loading | CEP lookup result stored locally |
| `CreditCardForm` | `{ cardholderName, stripeError }` | Stripe Elements state is Stripe-managed (never in our state) |
| `BoletoDataForm` | `{ cpfCnpj }` + validation error | CPF/CNPJ check-digit validated client-side |
| `PixQrModal` | `{ secondsRemaining, copied }` | Timer managed locally; `secondsRemaining = 0` triggers expiry state |

### 2.3 State Persistence

| Data | Storage | TTL | Notes |
|------|---------|-----|-------|
| `checkoutSessionId` | `sessionStorage` | Session lifetime | Restored on page reload; prevents new idempotency key on refresh |
| `activeStep` + `completedSteps` | `sessionStorage` | Session lifetime | Enables reload recovery to last step |
| Confirmed step data (contact, address) | `sessionStorage` | Session lifetime | Pre-fills review step; cleared on order confirmation |
| Payment fields | Never stored | — | Card data and CVV never written to any storage |

---

## 3. Step Transitions

### 3.1 Forward Transitions

```
[Summary] → [Contact] → [Address] → [Payment] → [Review] → [Submit] → [Confirmation]
```

- Each "Continue" click validates the current step; transition only occurs on clean validation.
- For returning users with a saved address and payment method, `ContactStep` and `AddressStep` may be pre-confirmed; flow starts at `[Review]`.

### 3.2 Backward Navigation

- "Edit" links on `ReviewStep` navigate back to the target step without clearing subsequent data.
- The step router preserves state for all steps when navigating backward.

### 3.3 Payment Method Branching

```
[PaymentMethodSelector]
       │
  ┌────┼────────────┐
  ▼    ▼            ▼
[CreditCardForm] [PixSummaryConfirm] [BoletoDataForm]
  │    │            │
  └────┴────────────┘
       ▼
  [ReviewStep]
```

- Selecting a different payment method unmounts the previous sub-component and mounts the new one.
- Switching methods clears the previous method's local state (no card data leak across sessions).

---

## 4. Checkout Mode Integration — Hosted Flow

Under the `hosted` checkout mode (as configured in `squad-config.json`), the checkout is entirely rendered inside a first-party page (not an offsite redirect). This means:

- The component tree above is the full implementation surface.
- Stripe.js Elements are embedded inline within `CreditCardForm`.
- Stripe's `confirmCardPayment` is the primary payment confirmation call.
- Redirect returns (3DS) are handled via a `return_url` pointing back to `/checkout/return` which reads the `payment_intent` query parameter and advances to `ConfirmationStep`.

---

## 5. Component Communication

| Pattern | Usage |
|---------|-------|
| Props drilling | Parent-to-child config (step number, static labels) |
| Callback props | Child-to-parent events (step confirmed, error raised) |
| Context | `CheckoutContext` exposes `sessionId`, `isSubmitting`, `globalError` to all descendant components |
| Direct state | Each step manages its own form state; no shared store for form fields |

---

## 6. Loading and Disabled States

- `isSubmitting = true` is set immediately when the submit request is dispatched and cleared only on terminal outcome (success, error, decline).
- All form inputs and the submit button receive `disabled` when `isSubmitting = true`.
- A `aria-live="polite"` region announces "Processing your order…" to screen readers when `isSubmitting` becomes `true`.
- The spinner within the submit button is CSS-based and respects `prefers-reduced-motion`.

---

## 7. Return URL and 3DS Recovery

When Stripe requires 3D Secure authentication:

1. `confirmCardPayment` resolves with a `next_action.redirect_to_url`.
2. The frontend navigates to that URL (Stripe-hosted 3DS page).
3. After authentication, Stripe redirects back to the `return_url` supplied on confirmation.
4. The `/checkout/return` handler reads `?payment_intent=pi_xxx&payment_intent_client_secret=...`.
5. It calls `stripe.retrievePaymentIntent(clientSecret)` to get the final status.
6. Based on `paymentIntent.status`:
   - `succeeded` → mount `CreditCardConfirmation` in success state.
   - `requires_payment_method` → return to `PaymentStep` with a soft decline error.
   - `processing` → poll until terminal; show "We're confirming your payment…" state.

---

## 8. Reload Recovery

On any page reload during an active checkout session:

1. `CheckoutSessionGuard` reads `checkoutSessionId` and `activeStep` from `sessionStorage`.
2. If a session exists and `orderResult` is not yet set, restore to the last confirmed step.
3. If `orderResult` already exists, redirect to `ConfirmationStep` without re-rendering the form.
4. Payment fields are never restored from storage — the user must re-enter payment details if they reload during the payment step.
