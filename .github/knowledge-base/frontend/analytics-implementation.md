# Analytics Implementation — Checkout Step and Outcome Instrumentation

**Phase 5 Deliverable — Frontend Checkout Implementation**
**PCI Scope**: SAQ-A | **Profile**: Hosted Checkout | **Currency**: BRL

---

## 1. Reference

Event schemas and SAQ-A compliance requirements are defined in [`ux/analytics-events.md`](../ux/analytics-events.md). This document covers the **implementation** of those schemas: where each event is fired, how the analytics layer is structured, and privacy enforcement.

---

## 2. Analytics Layer Architecture

### 2.1 Principle of Indirection

Checkout components must never call analytics APIs directly. All analytics calls go through a single `checkoutAnalytics` module:

```
CheckoutComponent
      │
      ▼  (calls domain method)
checkoutAnalytics.trackStepViewed(step)
      │
      ▼  (validates payload, strips any forbidden fields)
AnalyticsPrivacyFilter
      │
      ▼  (dispatches to configured backend or analytics client)
AnalyticsTransport
```

This ensures:
- A single place to enforce SAQ-A field exclusions.
- Easy swap of analytics backend (e.g., Segment, Mixpanel, custom endpoint) without changing component code.
- All events can be tested by inspecting `checkoutAnalytics` calls, not raw network traffic.

### 2.2 `checkoutAnalytics` Interface

```
interface CheckoutAnalytics {
  trackStepViewed(step: CheckoutStep): void;
  trackPaymentMethodSelected(method: PaymentMethodType, previous: PaymentMethodType | null, isSaved: boolean): void;
  trackSubmitted(method: PaymentMethodType, hasPromo: boolean, installments: number | null, attempt: number): void;
  trackSuccess(method: PaymentMethodType, orderStatus: string, attempt: number, timeToSubmitSeconds: number): void;
  trackDeclined(declineCategory: DeclineCategory, method: PaymentMethodType, attempt: number): void;
  trackAbandoned(lastStep: CheckoutStep, abandonReason: AbandonReason, timeInCheckoutSeconds: number): void;
  trackError(errorType: ErrorType, step: CheckoutStep, attempt: number, httpStatus: number | null): void;
  trackPixQrDisplayed(expiresInSeconds: number): void;
  trackPixCodeCopied(): void;
  trackPixTimerExpired(timeSinceGenerationSeconds: number): void;
  trackPixPaymentConfirmed(timeToPaySeconds: number): void;
  trackPixRegenerated(regenerationCount: number): void;
  trackBoletoIssued(expiresAt: string): void;
  trackBoletoPdfDownloaded(): void;
  trackBoletoBarcodeCopied(): void;
}
```

---

## 3. Base Properties Injection

All events automatically include the mandatory base properties defined in `analytics-events.md`. These are injected by the `checkoutAnalytics` module and never need to be passed by callers:

| Property | Source |
|----------|--------|
| `order_id` | `CheckoutContext.orderId` (set after backend confirms intent) |
| `session_id` | `CheckoutContext.checkoutSessionId` |
| `payment_method_type` | `CheckoutContext.selectedMethod` |
| `currency` | Static: `"BRL"` |
| `environment` | Build-time env variable (`VITE_ENV` / `NEXT_PUBLIC_ENV`) |
| `checkout_mode` | Static: `"hosted"` |
| `timestamp_utc` | `new Date().toISOString()` (captured at call time) |
| `user_type` | `CheckoutContext.userType` (`"guest"` / `"returning"`) |

---

## 4. Firing Points per Component

### 4.1 `CheckoutStepRouter`

```
// On every step mount / transition
checkoutAnalytics.trackStepViewed({
  stepName: currentStep,
  stepNumber: stepIndex,
  totalSteps: TOTAL_STEPS,
  isReturningUser: userType === 'returning',
});
```

Called on every step mount — including when navigating backward to a completed step.

### 4.2 `PaymentMethodSelector`

```
// On radio selection change
checkoutAnalytics.trackPaymentMethodSelected(
  newMethod,
  previousMethod,    // null if first selection
  isSavedMethod,
);
```

### 4.3 `CheckoutApp` — Submit Handler

```
// Immediately before the API call, after client-side validation passes
checkoutAnalytics.trackSubmitted(
  selectedMethod,
  hasPromoCode,
  installments,   // null for pix/boleto
  attemptNumber,
);
```

### 4.4 `CheckoutApp` — Confirmation Handler

```
// After backend confirms order
checkoutAnalytics.trackSuccess(
  selectedMethod,
  orderResult.status,          // "confirmed" | "pending_payment"
  attemptNumber,
  secondsSinceCheckoutEntry,
);
```

### 4.5 `CheckoutApp` — Decline Handler

```
// After backend returns decline_category
checkoutAnalytics.trackDeclined(
  backendResponse.declineCategory,   // mapped category, never raw Stripe code
  selectedMethod,
  attemptNumber,
);
```

### 4.6 `CheckoutApp` — Error Handler

```
// On system error (5xx, network timeout)
checkoutAnalytics.trackError(
  mapErrorToType(error),     // "network_timeout" | "server_error" | "tokenization_failed"
  currentStep,
  attemptNumber,
  error.httpStatus ?? null,
);
```

### 4.7 Abandonment — `beforeunload` Listener

```
window.addEventListener('beforeunload', () => {
  if (!orderResult && activeStep !== 'confirmation') {
    checkoutAnalytics.trackAbandoned(
      activeStep,
      'beforeunload',
      secondsSinceCheckoutEntry,
    );
  }
});
```

Also fires when user clicks "Return to cart":

```
checkoutAnalytics.trackAbandoned(activeStep, 'user_action', secondsSinceCheckoutEntry);
```

### 4.8 `PixQrModal`

```
// On modal mount
checkoutAnalytics.trackPixQrDisplayed(secondsUntilExpiry);

// On copy button click
checkoutAnalytics.trackPixCodeCopied();

// When timer reaches zero
checkoutAnalytics.trackPixTimerExpired(PIX_EXPIRY_SECONDS);

// On backend webhook confirmation (polled or server-sent)
checkoutAnalytics.trackPixPaymentConfirmed(secondsSinceQrDisplayed);

// On regeneration
checkoutAnalytics.trackPixRegenerated(regenerationCount);
```

### 4.9 `BoletoIssuedScreen`

```
// On screen mount
checkoutAnalytics.trackBoletoIssued(boletoData.expiresAt);

// On PDF download click
checkoutAnalytics.trackBoletoPdfDownloaded();

// On barcode copy click
checkoutAnalytics.trackBoletoBarcodeCopied();
```

---

## 5. Privacy Enforcement — SAQ-A Gate

The `AnalyticsPrivacyFilter` runs before every event dispatch. It scans all event properties and blocks the event if any forbidden field is found:

### 5.1 Blocked Field Patterns

| Pattern | Why |
|---------|-----|
| Any key containing `card`, `pan`, `number` | May contain PAN |
| Any key containing `cvv`, `cvc`, `security_code` | Card security code |
| Any key containing `expir` | Card expiry |
| Any key containing `stripe_token`, `payment_method_id`, `pm_` | Stripe token values |
| Any key containing `payment_intent`, `pi_` | Stripe intent IDs |
| Any string value matching `/\b\d{13,19}\b/` | PAN-like digit sequence |
| Any string value matching `/sk_(live|test)_/` | Stripe secret key pattern |

### 5.2 Violation Handling

- If a forbidden field is detected: block the event, log a warning to the application error tracker (without the offending value), and increment a `analytics_privacy_violation` counter.
- Do not silently discard violations — they should surface in monitoring dashboards.

---

## 6. SAQ-A Compliance Verification Checklist

Before going live, verify the following against the complete analytics event log:

- [ ] No event payload contains card number, expiry, CVV, or cardholder name (from payment step)
- [ ] No event payload contains a Stripe `pm_xxx` or `pi_xxx` identifier
- [ ] No event payload contains raw Stripe decline codes — only `decline_category` values
- [ ] `order_id` is an internal identifier, not a Stripe payment intent ID
- [ ] `session_id` is an anonymous UUID; no user PII is embedded in the session ID
- [ ] All analytics events are logged (not just the happy path) — including errors and abandonments
- [ ] The `AnalyticsPrivacyFilter` is exercised in the test suite with synthetic card data inputs to confirm blocking behavior (see Phase 6 test matrix)

---

## 7. Time Measurement

| Metric | Calculation |
|--------|------------|
| `time_to_submit_seconds` | `submitTimestamp - checkoutEntryTimestamp` |
| `time_in_checkout_seconds` | `abandonTimestamp - checkoutEntryTimestamp` |
| `time_to_pay_seconds` (Pix) | `pixConfirmedTimestamp - pixQrDisplayedTimestamp` |
| `time_since_generation_seconds` (Pix expiry) | Always `PIX_EXPIRY_SECONDS` constant (1800) |

- `checkoutEntryTimestamp` is set when `CheckoutApp` mounts and stored in the checkout context.
- All timestamps are UTC epoch milliseconds internally; converted to integer seconds for analytics payloads.
