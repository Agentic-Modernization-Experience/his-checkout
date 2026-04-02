# Data Classification Matrix — Checkout Payments

**Phase**: 2 — Security & Compliance

---

## 1. Classification Levels

| Level | Label | Description |
|---|---|---|
| L1 | **Restricted** | Cardholder data or credentials; must never appear in first-party systems |
| L2 | **Confidential** | Sensitive business or personal data; access-controlled, masked in logs |
| L3 | **Internal** | Non-public operational data; accessible to authorized services |
| L4 | **Public** | Safe to expose externally; no masking required |

---

## 2. Field-Level Classification

### Payment Fields

| Field | Classification | Storage | Logging | Transmission |
|---|---|---|---|---|
| Primary Account Number (PAN) | L1 — Restricted | Never in first-party systems | Never log | Stripe only (via hosted iframe) |
| CVV / CVC | L1 — Restricted | Never (including Stripe post-auth) | Never log | Stripe only; never stored |
| Expiry date | L1 — Restricted | Never in first-party systems | Never log | Stripe only |
| Cardholder name | L1 — Restricted | Never in first-party systems | Never log | Stripe only (optional) |
| PaymentMethod ID (`pm_xxx`) | L2 — Confidential | May be stored transiently per payment | Mask last 6 chars | Backend ↔ Stripe API only |
| PaymentIntent ID (`pi_xxx`) | L3 — Internal | Stored linked to order | May log | Backend ↔ Stripe API |
| Stripe Customer ID (`cus_xxx`) | L2 — Confidential | Stored linked to user | Mask prefix | Backend only |
| Payment status | L3 — Internal | Stored | May log | Backend → Client (sanitized) |
| Last-4 digits (display only) | L4 — Public | May store for display | May log | Client display |
| Card brand (display only) | L4 — Public | May store for display | May log | Client display |

### Order and User Fields

| Field | Classification | Storage | Logging | Transmission |
|---|---|---|---|---|
| Order ID | L3 — Internal | Stored | May log | Client, Backend, Stripe metadata |
| Customer email | L2 — Confidential | Stored | Mask domain in logs | Backend only |
| Customer full name | L2 — Confidential | Stored | Mask in logs | Backend only |
| Billing address | L2 — Confidential | Stored | Mask in logs | Backend ↔ Stripe |
| Shipping address | L2 — Confidential | Stored | Mask in logs | Backend → fulfillment |
| IP address | L2 — Confidential | Stored for fraud/audit | Mask in logs | Backend only |
| User agent | L3 — Internal | Stored for fraud/audit | May log | Backend only |
| Order total / amount | L3 — Internal | Stored | May log | Client, Backend |
| Currency | L4 — Public | Stored | May log | Client, Backend, Stripe |

### Webhook and Secret Fields

| Field | Classification | Storage | Logging | Transmission |
|---|---|---|---|---|
| `STRIPE_SECRET_KEY` | L1 — Restricted | Secrets manager / env var | Never log | Backend → Stripe API |
| `STRIPE_WEBHOOK_SECRET` | L1 — Restricted | Secrets manager / env var | Never log | Backend (local verification only) |
| Stripe event payload (raw) | L2 — Confidential | Transient (verify then discard raw body) | Partial (IDs only) | Stripe → Backend |
| Idempotency key | L3 — Internal | Stored per request | May log | Backend → Stripe API |

---

## 3. Masking Rules

| Data Type | Masking Pattern | Example |
|---|---|---|
| PAN (should not appear) | Never present; if leaked: `**** **** **** NNNN` | `**** **** **** 4242` |
| PaymentMethod ID | First 3 + `***` + last 4 | `pm_***xyz1` |
| Email address | Local part masked | `j***@example.com` |
| Full name | First initial + `***` | `J***` |
| Billing/Shipping address | Street masked, city/country allowed | `*** Main St, São Paulo, BR` |
| IP address | Last octet masked | `192.168.1.***` |
| Secret keys | Never appear in logs or responses | — |

---

## 4. Retention Rules

| Data Category | Retention Period | Deletion Trigger |
|---|---|---|
| Order records | 7 years (regulatory) | Manual purge after retention period |
| Payment event log | 13 months (chargeback window + buffer) | Automated purge |
| Webhook event IDs (deduplication store) | 30 days | Automated purge — daily scheduled job (`DELETE WHERE received_at < NOW() - INTERVAL '30 days'`); see `webhook-verification.md` |
| Idempotency keys | 30 days (matches Stripe's key retention) | Automated purge — daily scheduled job on `idempotency_keys.created_at` |
| Customer email / name | Until account deletion requested | GDPR/LGPD erasure request |
| Stripe `pm_xxx` tokens (transient) | Deleted after payment confirmation or failure | Payment lifecycle end |
| Fraud/audit logs (IP, user agent) | 13 months | Automated purge — daily scheduled job on `audit_logs.created_at` |

> **Implementation note**: Automated purge jobs are a backend implementation requirement for Phase 4 (Backend Payment Orchestration). The retention periods defined here are the authoritative requirements that the implementation must satisfy.

---

## 5. Data Handling Prohibitions

The following are strictly prohibited in first-party systems:

- Storing, logging, or transmitting PAN, CVV, expiry, or cardholder name
- Including raw Stripe secret keys in source code, issue bodies, or PR descriptions
- Forwarding raw Stripe API error objects to client responses (map to safe codes first)
- Persisting unverified webhook payloads to the order state database
