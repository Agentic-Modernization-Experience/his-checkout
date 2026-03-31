# Analytics Events Design

**Phase 1 Deliverable — Checkout UX & Flow Design**
**Profile**: Hosted Checkout | **PCI Scope**: SAQ-A | **Currency**: BRL

---

## 1. Guiding Principles

- **No sensitive payment data** in any event. No card numbers, CVV, expiry dates, full PANs, or tokens that could reconstruct card data.
- All events are SAQ-A compliant: cardholder data never enters our analytics pipeline.
- Event property names are snake_case.
- Events fire client-side (browser) unless noted.
- Every event includes the **mandatory base properties** defined in section 2.

---

## 2. Mandatory Base Properties (All Events)

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `order_id` | string | Internal order reference (not payment intent ID in raw form) | `"ord_abc123"` |
| `session_id` | string | Anonymous checkout session UUID | `"sess_xyz789"` |
| `payment_method_type` | string | Selected method at time of event | `"credit_card"` / `"pix"` / `"boleto"` |
| `currency` | string | ISO 4217 | `"BRL"` |
| `environment` | string | Deployment environment | `"production"` / `"staging"` |
| `checkout_mode` | string | Hosting mode | `"hosted"` |
| `timestamp_utc` | string | ISO 8601 UTC timestamp | `"2026-03-31T01:00:00Z"` |
| `user_type` | string | Authentication state at event time | `"guest"` / `"returning"` |

**Excluded from all events**: card number, expiry, CVV, full name (on payment step), bank account details, Stripe payment intent ID (use internal `order_id` only).

---

## 3. Checkout Step View Events

Fire on every checkout step render (page load or step transition).

### Event: `checkout_step_viewed`

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `step_name` | string | Human-readable step identifier | `"contact_info"` / `"payment_method"` / `"order_review"` |
| `step_number` | integer | Step index (1-based) | `3` |
| `total_steps` | integer | Total steps in current flow | `5` |
| `is_returning_user` | boolean | Whether user has saved methods | `false` |

**Fires on**: Contact Info, Address, Payment Method Selection, Card Form, Order Review, Confirmation.

---

## 4. Payment Method Selection Event

### Event: `payment_method_selected`

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `method` | string | Selected method | `"credit_card"` / `"pix"` / `"boleto"` |
| `previous_method` | string \| null | Previously selected method (if changed) | `"pix"` |
| `is_saved_method` | boolean | Whether a saved/returning-user method was selected | `false` |

---

## 5. Submission Events

### Event: `checkout_submitted`

Fires when the user clicks the submit CTA and client-side validation passes.

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `payment_method_type` | string | As in base | `"credit_card"` |
| `has_promo_code` | boolean | Whether a promo code was applied | `false` |
| `installments` | integer \| null | Number of installments selected (card only) | `1` |
| `attempt_number` | integer | Retry attempt count (1 = first try) | `1` |

---

## 6. Success Event

### Event: `checkout_success`

Fires after backend confirms payment authorization/creation.

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `payment_method_type` | string | As in base | `"boleto"` |
| `order_status` | string | Resulting order status | `"confirmed"` / `"pending_payment"` |
| `attempt_number` | integer | Which attempt succeeded | `1` |
| `time_to_submit_seconds` | integer | Seconds from step 1 view to submit | `120` |

**Note**: For Pix and Boleto, `order_status` will be `"pending_payment"` since confirmation is async.

---

## 7. Decline Event

### Event: `checkout_declined`

Fires when a payment authorization is declined.

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `decline_category` | string | Mapped category (not raw Stripe code) | `"insufficient_funds"` / `"expired_card"` / `"do_not_honor"` / `"generic"` |
| `attempt_number` | integer | Which attempt was declined | `1` |
| `payment_method_type` | string | As in base | `"credit_card"` |

**Mapping rule**: Raw Stripe decline codes are mapped server-side to `decline_category` values. Raw codes are never sent to the analytics client.

---

## 8. Abandonment Event

### Event: `checkout_abandoned`

Fires when a user navigates away from checkout without completing the flow. Triggered on `beforeunload` or explicit "Return to cart."

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `last_step` | string | Last step the user reached | `"payment_method"` |
| `last_step_number` | integer | Step index | `3` |
| `abandon_reason` | string | How we detected abandonment | `"beforeunload"` / `"user_action"` |
| `time_in_checkout_seconds` | integer | Seconds since checkout entry | `45` |

---

## 9. Error Events

### Event: `checkout_error`

Fires on system errors (network failures, 5xx responses).

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| `error_type` | string | Error classification | `"network_timeout"` / `"server_error"` / `"tokenization_failed"` |
| `step_name` | string | Step where error occurred | `"payment_method"` |
| `attempt_number` | integer | Which attempt triggered the error | `2` |
| `http_status` | integer \| null | HTTP status code if available | `503` |

**Excluded**: Error message text (may contain sensitive info), stack traces, Stripe error codes.

---

## 10. Pix-Specific Events

| Event | Trigger | Additional Properties |
|-------|---------|----------------------|
| `pix_qr_displayed` | Pix QR modal shown | `expires_in_seconds: 1800` |
| `pix_code_copied` | Copy code button clicked | — |
| `pix_timer_expired` | Countdown reaches 0 | `time_since_generation_seconds: 1800` |
| `pix_payment_confirmed` | Backend confirms Pix payment | `time_to_pay_seconds: 360` |
| `pix_regenerated` | New QR code generated after expiry | `regeneration_count: 1` |

---

## 11. Boleto-Specific Events

| Event | Trigger | Additional Properties |
|-------|---------|----------------------|
| `boleto_issued` | Boleto created and displayed | `expires_at: "2026-04-02"` |
| `boleto_pdf_downloaded` | Download PDF button clicked | — |
| `boleto_barcode_copied` | Copy barcode button clicked | — |

---

## 12. SAQ-A Compliance Verification

| Check | Rule |
|-------|------|
| Card number | Never appears in any event property |
| CVV | Never appears in any event property |
| Expiry date | Never appears in any event property |
| Cardholder name | Not sent on payment step events |
| Stripe PaymentIntent ID | Not sent to analytics; only `order_id` used |
| Stripe token | Never sent to analytics |
| Raw decline code | Mapped to `decline_category` server-side; raw code not forwarded |
| Analytics destination | Must use server-side event forwarding or client-side with no PAN in payload |

**Review gate**: Before Phase 5 implementation, event schemas must be reviewed against the SAQ-A requirement checklist to confirm zero cardholder data exposure.
