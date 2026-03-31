# Security Model — Checkout Payment Protection

**Phase**: 2 — Security & Compliance  
**Provider**: Stripe  
**Scope**: Hosted-field tokenization (SAQ-A)

---

## 1. Trust Boundaries

```
┌────────────────────────────────────────────────────────────────┐
│  Browser / Client                                              │
│  ┌────────────────────┐                                        │
│  │  Stripe.js         │  ← card data entered here only        │
│  │  (hosted fields)   │  ← never touches first-party JS       │
│  └────────┬───────────┘                                        │
│           │ PaymentMethod token (pm_xxx)                       │
└───────────┼────────────────────────────────────────────────────┘
            │ HTTPS / TLS 1.2+
┌───────────▼────────────────────────────────────────────────────┐
│  First-Party Backend                                           │
│  - Receives token only (no raw card data)                      │
│  - Creates PaymentIntent with idempotency key                  │
│  - Stores order state, NOT card data                           │
└───────────┬────────────────────────────────────────────────────┘
            │ Stripe SDK / TLS
┌───────────▼────────────────────────────────────────────────────┐
│  Stripe (PCI-DSS Level 1 certified)                            │
│  - Processes and stores card data                              │
│  - Sends signed webhooks to backend                            │
└────────────────────────────────────────────────────────────────┘
```

### Trust Levels

| Zone | Trust Level | What It May See |
|---|---|---|
| Browser (Stripe.js iframe) | Untrusted | Renders payment form; no card data in first-party scope |
| First-party backend | Partially trusted | Order data, Stripe token (`pm_xxx`), webhook events (after signature verification) |
| Stripe | Trusted (contractual) | Raw card data, payment methods, charge results |
| Webhook receiver | Zero trust until verified | Must verify `Stripe-Signature` header before acting |

---

## 2. Threat Model

### In-Scope Threats

| Threat | Attack Vector | Control |
|---|---|---|
| Card data interception | Network eavesdropping | TLS 1.2+ on all connections; Stripe.js isolates card data in iframe |
| Replay attack | Resend a captured payment request | Idempotency keys bound to order ID + attempt number |
| Duplicate order | Double-click or network retry | Server-side idempotency key checked before creating order or charge |
| Fake webhook | Attacker posts a crafted webhook | `Stripe-Signature` HMAC-SHA256 verified using `STRIPE_WEBHOOK_SECRET` |
| Webhook replay | Resend a valid, captured webhook | Event timestamp checked; duplicates stored and rejected |
| Unauthorized payment state mutation | Direct API call without auth | Payment-state transitions require authenticated backend session |
| Secrets leakage | Env var exposure in logs or errors | Keys never logged; masked in error responses |

### Out-of-Scope Threats

| Threat | Reason Out of Scope |
|---|---|
| Account takeover / authentication bypass | Handled by identity/auth layer (not in checkout scope) |
| SQL injection in non-payment endpoints | General application security concern; not payment-specific |
| DDoS against checkout endpoint | Infrastructure/WAF layer responsibility |

---

## 3. Security Controls Summary

| Control | Mechanism | Enforced Where |
|---|---|---|
| Card data isolation | Stripe.js hosted fields | Client (iframe origin: `js.stripe.com`) |
| Transport security | TLS 1.2+ required | Network / load balancer |
| Payment tokenization | `pm_xxx` token replaces PAN | Stripe → Backend handoff |
| Idempotency | Per-request key on all write operations | Backend (see `idempotency-keys` section) |
| Webhook authenticity | HMAC-SHA256 signature verification | Backend webhook handler |
| Webhook deduplication | Event ID stored and checked | Backend event store |
| Secret management | Environment variables, never hardcoded | Deployment environment |
| Audit logging | All payment transitions logged with order ID | Backend application log |
| Error masking | Stripe error codes mapped; raw messages not forwarded to client | Backend |

---

## 4. Data Flow — Card Capture

```
1. Client loads checkout page
2. Stripe.js initializes in an iframe (cross-origin; card data never in first-party DOM)
3. User enters card details → Stripe tokenizes → returns PaymentMethod ID (pm_xxx)
4. Client sends pm_xxx + order metadata to first-party backend (HTTPS POST)
5. Backend creates PaymentIntent via Stripe API (includes idempotency key)
6. Stripe confirms payment → sends signed webhook to backend
7. Backend verifies signature → updates order state → responds to client
```

Raw card data (PAN, CVV, expiry) never enters the first-party network at any step.

---

## 5. Security Assumptions

- Stripe is assumed PCI-DSS Level 1 compliant; their attestation covers card data handling.
- The first-party backend runs in an environment with secrets injected at runtime (not baked into images).
- TLS certificate validity is enforced and monitored at the infrastructure layer.
- The `STRIPE_WEBHOOK_SECRET` is unique per environment (production, staging, development).
- Any client that presents a `pm_xxx` token is still subject to backend authorization checks; the token alone is not proof of intent.
