# Accessibility Checklist

**Phase 1 Deliverable — Checkout UX & Flow Design**
**Standard**: WCAG 2.1 AA | **Profile**: Hosted Checkout | **Provider**: Stripe

---

## 1. Keyboard-Only Flow

### 1.1 Tab Order Requirements

- [ ] Tab order follows visual reading order (top → bottom, left → right) on all checkout steps.
- [ ] No focusable element is skipped or unreachable via keyboard alone.
- [ ] The "Skip to main content" link is the first focusable element on every page.
- [ ] Modal dialogs (Pix QR, Retry, Session Expired) trap focus within the modal while open.
- [ ] Closing a modal returns focus to the element that triggered it.
- [ ] Form submission via `Enter` key on the last field or the submit button triggers the same behavior as clicking.

### 1.2 Keyboard Interactions

| Component | Expected Key | Behavior |
|-----------|-------------|----------|
| Payment method radio group | Arrow keys | Moves selection between credit_card / Pix / Boleto |
| Installments select | Arrow keys / Enter | Opens select and moves between options |
| Submit button | Enter / Space | Submits form |
| "Copy code" button | Enter / Space | Copies to clipboard |
| Modal close button | Enter / Space / Escape | Closes modal; returns focus to trigger |
| "Edit section" link | Enter | Navigates to that checkout step |
| Promo code "Apply" | Enter | Submits promo code |
| CEP autocomplete | Tab after input | Moves to next field; autocomplete triggers on completion |

### 1.3 Focus Management

- [ ] After navigating to a new checkout step, focus moves to the step heading (`h2`) or first interactive element.
- [ ] After a validation error on submit, focus moves to the first field with an error.
- [ ] After a successful form submission (non-redirect step), focus moves to the confirmation message or next-step heading.
- [ ] After opening a modal, focus moves to the modal's first interactive element or the modal title.
- [ ] Spinner / loading overlays do not remove focus from the triggering element (announce loading state via `aria-live`).

---

## 2. Screen Reader Expectations

### 2.1 Semantic HTML

- [ ] All form fields use `<label>` elements associated via `for` / `id` (or `aria-label` / `aria-labelledby` for Stripe Elements).
- [ ] Error messages are associated with their field via `aria-describedby`.
- [ ] Required fields are marked with `aria-required="true"` (not just a visual asterisk).
- [ ] Page title (`<title>`) updates to reflect the current checkout step (e.g., "Payment Method — Checkout").
- [ ] Step indicators (e.g., "Step 2 of 4") use `aria-label` to provide full context (not just the visual number).

### 2.2 ARIA Live Regions

| Region | `aria-live` Level | When It Updates |
|--------|------------------|-----------------|
| Inline field error message | `assertive` | On field blur (error appears) |
| Form-level error summary | `assertive` | On failed form submission |
| Loading / processing state | `polite` | On submit start |
| Success message | `polite` | On step completion |
| Countdown timer (Pix) | `polite` (update every 60s, not every second) | Timer updates |
| Timer expired alert | `assertive` | When timer reaches 0 |
| "Code copied" toast | `polite` | On copy action |

### 2.3 Stripe Elements Accessibility

- [ ] Stripe Elements iframes expose `aria-label` attributes set via `options.style` or `placeholder`.
- [ ] Set explicit placeholder text for each Stripe element (Card number, MM/YY, CVC).
- [ ] Associate a visible label with each Stripe element container using `aria-labelledby` on the wrapper `<div>`.
- [ ] Test with NVDA + Chrome and VoiceOver + Safari to confirm field labels are announced.

---

## 3. Color and Contrast

- [ ] All text (labels, error messages, helper text) meets **WCAG AA contrast ratio of 4.5:1** against its background.
- [ ] Error state indicated by **both color and icon/text** (not color alone) — per WCAG 1.4.1.
- [ ] Active/focused input border meets 3:1 contrast ratio against adjacent background.
- [ ] Submit button in loading state maintains sufficient contrast (disabled visual ≠ invisible).
- [ ] Timer countdown in Pix modal: highlight urgency (< 5 min) without relying solely on color (add "⚠ Expiring soon" text).

---

## 4. Touch and Motor Accessibility

- [ ] All interactive elements (buttons, inputs, links) have a minimum touch target of **44 × 44 CSS pixels**.
- [ ] Adequate spacing between radio options (payment method list) to prevent accidental selection.
- [ ] Submit button is not placed immediately adjacent to destructive actions (e.g., "Cancel" / "Return to cart").
- [ ] "Copy code" and "Download PDF" buttons are large enough for users with motor difficulties.

---

## 5. Forms and Error Recovery

- [ ] Errors are described in text, not just by changing border color.
- [ ] Error summary at the top of the form (after failed submit) lists all errors with anchor links to each field.
- [ ] Error messages do not disappear without user action (they persist until the field is corrected).
- [ ] On successful correction of a field, error is cleared immediately on input or blur (before re-submit).
- [ ] Time limits (e.g., Pix 30-min timer) provide a way to extend or are clearly communicated in advance (WCAG 2.2.1).

---

## 6. Motion and Animation

- [ ] All animations (step transitions, loading spinners) respect `prefers-reduced-motion` media query.
- [ ] Pix countdown timer does not flash or animate in a way that could trigger photosensitivity issues.
- [ ] Spinner animations are CSS-based (not GIF) to allow `prefers-reduced-motion` suppression.

---

## 7. Language and Readability

- [ ] `<html lang="pt-BR">` set on all checkout pages.
- [ ] Error messages written at a reading level accessible to general population (avoid jargon).
- [ ] Abbreviations (CPF, CNPJ, CVV, CEP) have `<abbr title="...">` or are spelled out in context.

---

## 8. Testing Checklist

| Test | Tool / Method |
|------|--------------|
| Keyboard-only navigation | Manual — tab through entire flow without mouse |
| Screen reader — field labels | NVDA + Chrome; VoiceOver + Safari |
| Screen reader — error announcement | Test each error state individually |
| Screen reader — modal behavior | Verify focus trap and focus return |
| Color contrast — text | axe DevTools or Colour Contrast Analyser |
| Color contrast — interactive elements | Manual check for buttons and inputs |
| Zoom to 200% | Browser zoom — verify no horizontal scrolling or clipped content |
| Touch target size | Chrome DevTools device emulator + manual measurement |
| `prefers-reduced-motion` | Chrome DevTools → Rendering → Emulate CSS media feature |
| WCAG 2.1 AA automated scan | axe-core integration in CI (phase 6 testing) |
