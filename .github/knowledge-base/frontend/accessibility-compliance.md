# Accessibility Compliance — Frontend Validation Artifacts

**Phase 5 Deliverable — Frontend Checkout Implementation**
**Standard**: WCAG 2.1 AA | **Profile**: Hosted Checkout | **Provider**: Stripe | **Currency**: BRL

---

## 1. Reference

The accessibility requirements are specified in [`ux/accessibility-checklist.md`](../ux/accessibility-checklist.md). This document covers the **implementation** side: which component implements each requirement, what the validation artifact is, and what the acceptance criteria are for Phase 5.

---

## 2. Component-Level Implementation Map

### 2.1 `CheckoutProgress` — Step Indicator

| Requirement | Implementation |
|-------------|----------------|
| Step count announced to screen readers | `<nav aria-label="Checkout progress"><ol>` with `<li aria-current="step">` for the active step |
| Step labels not visual-only | Each `<li>` contains visible text label (e.g., "Payment") plus `aria-label="Step 2 of 5: Payment"` |
| Completed steps distinguishable | `aria-label="Step 1 of 5: Contact — completed"` on completed items |

**Validation**: NVDA + Chrome — confirm that advancing to a new step announces "Step N of 5: [Name]".

---

### 2.2 `CheckoutStepRouter` — Focus Management

| Requirement | Implementation |
|-------------|----------------|
| Focus moves to step heading on transition | After step mount, `stepHeadingRef.current.focus()` (the `<h2>`) |
| Step heading is focusable | `<h2 tabIndex={-1}>` — programmatically focusable but not in tab order |
| Loading state announced | `<div role="status" aria-live="polite">` updated to "Processing your order…" during `isSubmitting` |

**Validation**: Tab through the full flow with keyboard only; confirm that each step heading receives focus on transition.

---

### 2.3 Form Fields — All Steps

| Requirement | Implementation |
|-------------|----------------|
| Labels associated via `for`/`id` | Every `<input>` has a `<label htmlFor="fieldId">` |
| Required fields marked | `aria-required="true"` on all required inputs |
| Error messages linked to field | `aria-describedby="fieldId-error"` on the input; error `<span id="fieldId-error">` |
| Error messages announced | `<span role="alert">` on inline error elements |
| Error focus on failed submit | Focus moves to first invalid field after failed submit attempt |

**Validation**: Submit each step with all fields blank; confirm screen reader announces the first error message.

---

### 2.4 `PaymentMethodSelector` — Radio Group

| Requirement | Implementation |
|-------------|----------------|
| Radio group has label | `<fieldset><legend>Select payment method</legend>` wrapping all radio inputs |
| Keyboard navigation | Native `<input type="radio">` — arrow keys move selection; `Enter`/`Space` confirms |
| Selected state communicated | Native `checked` attribute; no custom ARIA needed |
| Icons not meaning-carrying alone | Method name text always present alongside icon |

**Validation**: Navigate the payment method list with arrow keys only; confirm all three methods are reachable and selectable.

---

### 2.5 `CreditCardForm` — Stripe Elements

| Requirement | Implementation |
|-------------|----------------|
| Labels for Stripe Elements | `<label id="card-number-label">Card number</label>` + `<div aria-labelledby="card-number-label">` wrapping the element |
| Placeholder text set | `elements.create('cardNumber', { placeholder: '1234 5678 9012 3456' })` |
| Errors associated | Stripe element error text placed in `<span role="alert" aria-live="assertive">` below each element |
| CVC field labeled | `<label id="cvc-label">Security code (CVV)</label>` + `<abbr title="Card Verification Value">CVV</abbr>` |

**Validation**: VoiceOver + Safari — confirm card field labels are announced on focus. NVDA + Chrome — confirm error announcement when an invalid card number is entered.

---

### 2.6 `PixQrModal` — Modal Focus Trap

| Requirement | Implementation |
|-------------|----------------|
| Modal announced on open | `role="dialog" aria-modal="true" aria-labelledby="pix-modal-title"` |
| Focus trapped inside modal | FocusTrap component (or equivalent) captures Tab/Shift+Tab within the modal bounds |
| Focus moves to modal on open | `firstFocusableElement.focus()` on modal mount |
| Focus returns on close | Ref to the trigger button is retained; focus returns to it on modal close |
| Dismiss via Escape | `keydown` listener: `Escape` → close modal; `event.stopPropagation()` to prevent page navigation |
| Countdown timer not overly verbose | `aria-live="polite"` with updates every 60 seconds (not every second); final 5-minute warning uses `aria-live="assertive"` |

**Validation**: Open Pix modal with keyboard; confirm focus is trapped. Close with Escape; confirm focus returns to the "Place Order" button.

---

### 2.7 `BoletoIssuedScreen`

| Requirement | Implementation |
|-------------|----------------|
| Barcode text readable | `<p aria-label="Boleto barcode number">` containing the barcode string |
| Copy button accessible | `<button aria-label="Copy barcode number">` — announces "Copied!" via `aria-live="polite"` region |
| PDF link is meaningful | `<a href="..." aria-label="Download boleto PDF (opens in new tab)" target="_blank" rel="noopener">` |
| Expiry prominently communicated | `<p><strong>Payment due by: <time dateTime="2026-04-04">April 4, 2026</time></strong></p>` |

**Validation**: Navigate the boleto screen with keyboard; confirm all actions are reachable and labeled.

---

### 2.8 Error and Retry Modal

| Requirement | Implementation |
|-------------|----------------|
| Error modal uses `role="alertdialog"` | `role="alertdialog" aria-labelledby="error-title" aria-describedby="error-message"` |
| Focus moves to modal | `errorModalRef.current.focus()` on mount |
| Retry button is primary focus | Retry button is the first focusable element in the modal |
| Focus returns after close | Returns to submit button on "Cancel" |

**Validation**: Trigger a 5xx error; confirm modal opens and focus is on the Retry button.

---

## 3. Keyboard Navigation — Complete Flow Verification

The following full flow must be completable with keyboard only (no mouse):

| Step | Keyboard Action | Expected Outcome |
|------|----------------|-----------------|
| 1. Arrive at Contact step | Tab into first field | "Full Name" field focused and announced |
| 2. Fill all contact fields | Tab + type | Each field reachable and input accepted |
| 3. Submit Contact step | Tab to "Continue", press Enter | Focus moves to Address step heading |
| 4. Enter CEP | Type 8 digits | Autocomplete triggers; fields filled |
| 5. Navigate address fields | Tab | All fields reachable |
| 6. Submit Address step | Enter on "Continue" | Focus moves to Payment step heading |
| 7. Select Pix | Arrow keys on radio group | "Pix" announced as selected |
| 8. Submit payment step | Enter on "Place Order" | `isSubmitting` announced; Pix QR modal opens |
| 9. Copy Pix code | Tab to copy button, Enter | "Copied" toast announced by `aria-live` |
| 10. Close modal | Escape | Focus returns to "Place Order" button |

---

## 4. ARIA Live Region Summary

| Region | Location | Level | When Updated |
|--------|----------|-------|-------------|
| `#checkout-status` | `CheckoutApp` root | `polite` | "Processing…" on submit; cleared on result |
| `#step-errors` | Each step form | `assertive` | On failed submit; cleared on success |
| `#pix-timer` | `PixQrModal` | `polite` | Every 60 seconds |
| `#pix-timer-urgent` | `PixQrModal` | `assertive` | When < 5 minutes remain |
| `#copy-toast` | Copy buttons | `polite` | On successful clipboard write |
| `#field-error-{id}` | Each form field | `assertive` (via `role="alert"`) | On field error change |

---

## 5. Automated Scan Gate (Pre-Launch)

Before any production release, the following automated checks must pass:

| Tool | Scope | Pass Criteria |
|------|-------|--------------|
| axe-core (via jest-axe or Playwright) | All checkout step renders | Zero violations at `critical` and `serious` levels |
| eslint-plugin-jsx-a11y | All JSX components | Zero lint warnings in CI |
| Lighthouse accessibility audit | Rendered checkout pages | Score ≥ 90 |

These are defined as acceptance criteria for Phase 6 (Testing & Payment Simulation).

---

## 6. Manual Testing Matrix

| Test | Tool | Tester |
|------|------|--------|
| Full keyboard-only flow (guest checkout) | Keyboard only | QA + Accessibility reviewer |
| Screen reader — field labels and errors | NVDA + Chrome | Accessibility reviewer |
| Screen reader — modal focus trap | VoiceOver + Safari | Accessibility reviewer |
| Color contrast — all states | Colour Contrast Analyser | Designer |
| 200% browser zoom — no horizontal overflow | Chrome zoom | QA |
| `prefers-reduced-motion` — no animations | Chrome DevTools | QA |
| Touch target minimum 44px | Mobile device / DevTools emulator | QA |
| Screen reader — Pix countdown timer | NVDA | Accessibility reviewer |

Results from manual testing must be documented and signed off before the Phase 5 PR is merged.
