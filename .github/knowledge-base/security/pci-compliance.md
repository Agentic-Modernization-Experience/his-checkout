# PCI DSS Compliance Notes — Checkout

**Phase**: 2 — Security & Compliance  
**SAQ Level**: SAQ-A  
**Provider**: Stripe (PCI-DSS Level 1 certified)

---

## 1. Scope Definition

This checkout implementation uses **Stripe.js hosted fields** (embedded iframe). All cardholder data is entered directly into Stripe-controlled iframes and never touches first-party JavaScript, servers, or storage.

This approach qualifies for **SAQ-A**, the smallest PCI DSS scope available to merchants.

### Scope Boundaries

| Component | In PCI Scope | Reason |
|---|---|---|
| Stripe.js iframe (js.stripe.com) | No (Stripe's scope) | Cross-origin iframe; Stripe is the merchant of record for card capture |
| First-party frontend JavaScript | No | Card data never present in first-party DOM or network requests |
| First-party backend | No | Receives only `pm_xxx` token; no PAN, CVV, or expiry |
| Database / storage | No | Token stored, not card data |
| Stripe servers | Yes (Stripe's scope) | Full card data stored and processed by Stripe |

---

## 2. SAQ-A Requirements Summary

SAQ-A applies to merchants that fully outsource card capture to a PCI-compliant third party and have no electronic storage, processing, or transmission of cardholder data on their own systems.

| SAQ-A Requirement | Compliance Approach |
|---|---|
| Req 2 — Secure default configurations | Stripe.js served from Stripe CDN; no configuration of card-capture infrastructure required |
| Req 6 — Secure systems and applications | Application code does not handle card data; standard secure development applies |
| Req 8 — Identify and authenticate access | Backend access controls and authentication are standard application responsibilities |
| Req 9 — Restrict physical access | Not applicable; no on-premises cardholder data systems |
| Req 12 — Information security policies | Maintain a policy acknowledging the hosted tokenization model and merchant responsibilities |

---

## 3. Cardholder Data Handling Boundaries

```
CLIENT                        FIRST-PARTY BACKEND           STRIPE
──────                        ───────────────────           ──────
Card number                   ✗ never received              ✓ held in Stripe vault
CVV / CVC                     ✗ never received              ✓ processed, not stored
Expiry date                   ✗ never received              ✓ stored in Stripe vault
Cardholder name               ✗ never received              ✓ optional, Stripe-only
PaymentMethod ID (pm_xxx)     ✓ passed to backend           ✓ created by Stripe
Order ID                      ✓ created by backend          ✓ attached as metadata
```

**No cardholder data fields may ever be transmitted to or stored in first-party systems.**

---

## 4. Tokenization Approach

- **Mechanism**: Stripe.js `stripe.createPaymentMethod()` or Elements flow
- **Token type**: `PaymentMethod` object (`pm_xxx`) returned to first-party code
- **Token usage**: Backend passes `pm_xxx` to `PaymentIntents.create()` or `confirm()`
- **Token storage**: Only `pm_xxx` and `PaymentIntent` ID (`pi_xxx`) may be persisted
- **Reuse policy**: `pm_xxx` tokens are single-use per payment flow unless explicitly saved to a `Customer` object (requires explicit user consent)

---

## 5. Annual SAQ-A Self-Assessment Checklist

- [ ] Confirm Stripe.js is loaded from `https://js.stripe.com` only
- [ ] Verify no card-field values are ever read by first-party JavaScript
- [ ] Confirm no cardholder data appears in application logs
- [ ] Review and renew Stripe's current PCI DSS Attestation of Compliance (AoC)
- [ ] Confirm `STRIPE_WEBHOOK_SECRET` is rotated annually or upon suspected compromise
- [ ] Confirm the hosted-payment integration model has not changed (e.g., no custom card input fields added)

---

## 6. Key Responsibilities

| Responsibility | Owner |
|---|---|
| Card data security and PCI Level 1 certification | Stripe |
| Not introducing first-party card data handling | Engineering team |
| Keeping Stripe.js loaded from official CDN | Engineering team |
| Annual SAQ-A self-assessment | Compliance / Engineering |
| Reviewing Stripe AoC annually | Compliance |
