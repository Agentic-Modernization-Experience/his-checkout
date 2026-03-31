# Responsive Behavior Notes

**Phase 1 Deliverable — Checkout UX & Flow Design**
**Profile**: Hosted Checkout | **Breakpoints**: Mobile-first | **Currency**: BRL

---

## 1. Breakpoint Definitions

| Breakpoint | Range | Layout Target |
|-----------|-------|--------------|
| `xs` | < 480px | Single-column; compact form |
| `sm` | 480px – 767px | Single-column; slightly relaxed spacing |
| `md` | 768px – 1023px | Two-column optional (form + summary) |
| `lg` | ≥ 1024px | Two-column preferred (form left, summary right) |

---

## 2. Layout Behavior by Breakpoint

### 2.1 Checkout Container

| State | Mobile (xs/sm) | Desktop (lg+) |
|-------|---------------|---------------|
| Layout | Single column; full-width | Two-column: form 60% / summary 40% |
| Order Summary | Collapsed by default (expandable) | Always visible in right column |
| Step indicators | Horizontal pills, text abbreviated | Full labels (e.g., "Payment Method") |
| Back / navigation | Sticky bottom bar | Inline link at top of section |

### 2.2 Form Fields

| State | Mobile (xs/sm) | Desktop (lg+) |
|-------|---------------|---------------|
| Field width | 100% of container | Max 480px for most fields; full-width for address |
| Two-field rows | Stack vertically (e.g., City + State) | Side-by-side (City 70% / State 30%) |
| Keyboard trigger | Native mobile keyboard; `inputmode` attribute set per field | Standard keyboard |
| Label position | Above field (stacked) | Above field (stacked) — consistent across breakpoints |

**`inputmode` mapping**:
- CEP: `inputmode="numeric"`
- Phone: `inputmode="tel"`
- Card Number: `inputmode="numeric"` (Stripe-managed)
- CPF/CNPJ: `inputmode="numeric"`
- Email: `inputmode="email"`

---

## 3. Payment Method Selection

| State | Mobile | Desktop |
|-------|--------|---------|
| Method list | Vertical card list; full-width tap targets | Vertical list or 3-column grid |
| Selected state | Clear checkmark + border highlight | Same |
| Method icons | Prominent (≥ 48px) | Same (≥ 48px) |
| Description text | Shown below selected method only | Always visible below each option |

---

## 4. Credit Card Form

| Element | Mobile | Desktop |
|---------|--------|---------|
| Card Number | Full-width | Full-width (up to 480px) |
| Expiry + CVV | Side-by-side (50/50) | Side-by-side (50/50) |
| Cardholder Name | Full-width | Full-width |
| Stripe Elements height | 44px minimum (touch target) | 44px minimum |

---

## 5. Pix QR Modal

| Element | Mobile | Desktop |
|---------|--------|---------|
| Modal size | Full-screen bottom sheet (slide-up) | Centered dialog (max-width 480px) |
| QR code size | 200 × 200px minimum | 240 × 240px |
| Copy code button | Full-width; below QR | Full-width; below QR |
| Timer | Prominent; top of content area | Prominent; top of content area |
| Dismiss control | "×" button top-right; pull-to-dismiss | "×" button; Escape key |

---

## 6. Boleto Result Screen

| Element | Mobile | Desktop |
|---------|--------|---------|
| Layout | Single column; stacked | Single column (centered, max 600px) |
| Barcode text | Scrollable horizontal text area | Same, with copy button |
| Download button | Full-width primary CTA | Standard width button |
| Expiry notice | Bold, above fold | Same |

---

## 7. Error and Retry Modal

| Element | Mobile | Desktop |
|---------|--------|---------|
| Modal size | Full-screen bottom sheet | Centered dialog (max-width 420px) |
| Message | Large, readable (16px+) | Same |
| CTAs | Stacked: "Retry" (primary) / "Cancel" (secondary) | Side-by-side or stacked |
| Background overlay | Full-screen dark overlay | Semi-transparent overlay |

---

## 8. Submit Button Behavior

| State | Mobile | Desktop |
|-------|--------|---------|
| Default | Fixed/sticky at bottom of viewport during form fill | Inline at bottom of form |
| Loading | Shows spinner in button; button full-width sticky | Spinner inline; button greyed |
| After success | Hidden (redirect) | Hidden (redirect) |

**Note**: Sticky submit on mobile prevents the user from scrolling away from the CTA. The button must always remain visible during form entry on small screens.

---

## 9. Order Summary (Collapsed on Mobile)

| State | Mobile | Desktop |
|-------|--------|---------|
| Default | Collapsed; shows total only | Full summary in right column |
| Expanded | Slides down; shows full item list | Always expanded |
| Toggle | Tap "Order Summary ▼" | Not applicable |
| Position | Below step progress bar | Fixed right column (sticky) |

---

## 10. Typography and Spacing Scale

| Token | Mobile | Desktop |
|-------|--------|---------|
| Base font size | 16px (never below; prevents iOS zoom) | 16px |
| Form label | 14px medium | 14px medium |
| Input text | 16px regular | 16px regular |
| Error text | 13px regular; error color | 13px regular |
| H2 (step title) | 18px semibold | 20px semibold |
| CTA button text | 16px semibold | 16px semibold |
| Section spacing | 24px between sections | 32px between sections |
| Field margin-bottom | 16px | 20px |

**Important**: Minimum font size for inputs is 16px on iOS to prevent automatic zoom when focusing a form field.

---

## 11. Confirmation Page

| Element | Mobile | Desktop |
|---------|--------|---------|
| Layout | Single column; centered content | Centered single column (max 600px) |
| Order number | Large; prominent | Large; prominent |
| Success icon | ≥ 64px | ≥ 80px |
| Continue shopping CTA | Full-width | Standard width |
| Payment method summary | Concise; one line | Concise; one line |
