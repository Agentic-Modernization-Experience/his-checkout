# Stripe SDK Integration — Tokenization Boundary

**Phase 5 Deliverable — Frontend Checkout Implementation**
**Provider**: Stripe | **PCI Scope**: SAQ-A | **Checkout Mode**: Hosted | **Currency**: BRL

---

## 1. Integration Principle

The cardinal rule of this integration is: **no raw cardholder data (PAN, CVV, expiry) ever enters first-party JavaScript, network requests, or storage.**

Stripe.js achieves this by rendering card input fields inside cross-origin iframes (`js.stripe.com`). First-party code only receives a `PaymentMethod` token (`pm_xxx`) or a `PaymentIntent` result — neither contains cardholder data.

---

## 2. Stripe.js Initialization

```
<!-- Loaded from Stripe CDN only — no self-hosting or bundling -->
<script src="https://js.stripe.com/v3/" async></script>
```

- The script must be loaded from `https://js.stripe.com` exclusively.
- Never bundle Stripe.js into the application bundle — this would break the SAQ-A boundary.
- Initialize once at application boot using the **publishable key** (public, safe for frontend):

```
const stripe = Stripe(STRIPE_PUBLISHABLE_KEY);
```

- `STRIPE_PUBLISHABLE_KEY` is a non-secret frontend environment variable (`pk_live_...` or `pk_test_...`).
- The **secret key** (`sk_...`) is never present in frontend code, environment variables, or source.

---

## 3. Stripe Elements Setup

Elements are created once per checkout session and reused across renders:

```
const elements = stripe.elements({
  locale: 'pt-BR',
  fonts: [{ cssSrc: '...' }],
});
```

### 3.1 Element Instances

| Element | Stripe Element Type | Mounted In |
|---------|-------------------|------------|
| Card Number | `CardNumberElement` | `#stripe-card-number` |
| Expiry Date | `CardExpiryElement` | `#stripe-card-expiry` |
| CVC / CVV | `CardCvcElement` | `#stripe-card-cvc` |

```
const cardNumberElement = elements.create('cardNumber', {
  placeholder: '1234 5678 9012 3456',
  style: { base: { fontSize: '16px', color: '#1a1a1a' } },
});
cardNumberElement.mount('#stripe-card-number');
```

- Each element must have an associated visible `<label>` for accessibility (see `accessibility-compliance.md`).
- Apply `aria-labelledby` on each element's container `<div>` pointing to the label `id`.

### 3.2 Element Event Listeners

```
cardNumberElement.on('change', (event) => {
  if (event.error) {
    setStripeFieldError(event.error.message);  // display inline error
  } else {
    setStripeFieldError(null);                 // clear inline error
  }
  setCardComplete(event.complete);             // enable submit when all fields complete
});
```

- Never read or log `event.value` — it does not contain sensitive data by design, but the rule prevents accidental logging if the API changes.
- Respond to `error` and `complete` flags only.

---

## 4. Payment Confirmation Flow

### 4.1 Credit Card — `confirmCardPayment`

**Step 1**: Backend creates a `PaymentIntent` and returns `clientSecret` to the frontend.

```
POST /api/checkout/confirm
Body: { orderId, idempotencyKey, paymentMethodType: "credit_card" }
Response: { clientSecret: "pi_xxx_secret_yyy" }
```

**Step 2**: Frontend calls Stripe to confirm the payment:

```
const result = await stripe.confirmCardPayment(clientSecret, {
  payment_method: {
    card: cardNumberElement,
    billing_details: {
      name: cardholderName,           // from our DOM input
      email: customerEmail,           // from ContactStep state
    },
  },
  return_url: `${window.location.origin}/checkout/return`,
});
```

**Step 3**: Handle result:

```
if (result.error) {
  // Tokenization or authorization failure
  handlePaymentError(result.error);
} else if (result.paymentIntent.status === 'succeeded') {
  handlePaymentSuccess(result.paymentIntent);
} else if (result.paymentIntent.status === 'requires_action') {
  // 3DS redirect handled automatically by Stripe via return_url
}
```

### 4.2 Pix — Server-Side Charge

Pix does not use Stripe Elements. The payment source is created entirely server-side:

```
POST /api/checkout/confirm
Body: { orderId, idempotencyKey, paymentMethodType: "pix" }
Response: {
  orderId,
  status: "pending_payment",
  pix: {
    qrCodeBase64: "...",
    copyPasteCode: "00020126...",
    expiresAt: "2026-04-02T15:30:00Z"
  }
}
```

Frontend renders the QR modal with the `pix` payload. No Stripe.js call is required on the frontend for Pix.

### 4.3 Boleto — Server-Side Charge

Similar to Pix — the Boleto is created server-side using the CPF/CNPJ provided by the user:

```
POST /api/checkout/confirm
Body: { orderId, idempotencyKey, paymentMethodType: "boleto", taxId: "cpfOrCnpj" }
Response: {
  orderId,
  status: "pending_payment",
  boleto: {
    barcode: "03399.33335...",
    pdfUrl: "https://pay.stripe.com/boleto/...",
    expiresAt: "2026-04-04"
  }
}
```

Frontend renders the boleto result screen. No Stripe.js call required.

---

## 5. Wrapper Contract

A `StripeProvider` wrapper centralizes initialization and exposes a typed API surface to the rest of the application:

```
interface StripeProvider {
  // Element instances (created once, reused across re-renders)
  cardNumberElement: StripeCardNumberElement;
  cardExpiryElement: StripeCardExpiryElement;
  cardCvcElement:    StripeCardCvcElement;

  // Confirmation methods — only these are exposed to checkout components
  confirmCardPayment(clientSecret: string, billingDetails: BillingDetails): Promise<CardPaymentResult>;

  // Lifecycle
  destroy(): void; // called on checkout unmount to clean up elements
}
```

- No other Stripe SDK methods are exposed outside `StripeProvider`.
- Provider-specific error types (`StripeError`) are mapped to `ClientSafeError` before propagating out of the provider layer.
- `StripeProvider` is the only location where `stripe.*` methods are called.

---

## 6. Security Boundaries — Enforcement Rules

| Rule | Enforcement |
|------|-------------|
| Stripe.js loaded from CDN only | Content-Security-Policy: `script-src https://js.stripe.com` |
| Stripe iframes allowed | Content-Security-Policy: `frame-src https://js.stripe.com https://hooks.stripe.com` |
| No card field values in application code | Code review gate; ESLint rule prohibiting access to `CardElement` value properties |
| Publishable key only in frontend env | Secret scanning blocks `sk_live_` / `sk_test_` patterns in source and CI |
| No cardholder data in application logs | Logging middleware strips any field matching `card`, `cvv`, `cvc`, `expiry`, `pan` |
| No cardholder data in analytics | See `analytics-implementation.md` — event schemas exclude all card fields |

---

## 7. 3DS and Challenge Flows

When `paymentIntent.status === 'requires_action'` and `next_action.type === 'use_stripe_sdk'`, Stripe.js handles the redirect automatically via the `return_url` provided to `confirmCardPayment`.

**Return URL handler** (`/checkout/return`):

```
// On mount, read URL params and retrieve intent status
const params = new URLSearchParams(window.location.search);
const clientSecret = params.get('payment_intent_client_secret');

const { paymentIntent, error } = await stripe.retrievePaymentIntent(clientSecret);

switch (paymentIntent?.status) {
  case 'succeeded':
    advanceToConfirmation({ success: true });
    break;
  case 'requires_payment_method':
    returnToPaymentStep({ error: 'card_declined' });
    break;
  case 'processing':
    startPollingForCompletion(paymentIntent.id);
    break;
  default:
    advanceToConfirmation({ error: 'unknown_state' });
}
```

- The `clientSecret` is **not** stored after use; it is read once from the URL and discarded.
- The URL is replaced with `history.replaceState` to remove the `payment_intent_client_secret` from the browser history and prevent it from being shared or bookmarked.

---

## 8. Cleanup on Unmount

When the checkout component unmounts (navigation away, session end):

```
stripeProvider.destroy(); // Calls element.unmount() on all mounted Elements
```

This prevents memory leaks and ensures that Stripe iframes are fully removed from the DOM.
