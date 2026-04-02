# Secret Handling Rules — Checkout Payments

**Phase**: 2 — Security & Compliance  
**Provider**: Stripe

---

## 1. Secrets Inventory

| Secret Name | Purpose | Who Uses It |
|---|---|---|
| `STRIPE_SECRET_KEY` | Authenticate server-side Stripe API calls | Backend only |
| `STRIPE_PUBLISHABLE_KEY` | Initialize Stripe.js in the browser | Frontend (safe to expose publicly) |
| `STRIPE_WEBHOOK_SECRET` | Verify webhook signature (`whsec_xxx`) | Backend webhook handler |

> `STRIPE_PUBLISHABLE_KEY` is not a secret — it is designed to be public. It is listed here for inventory completeness only. It must never be confused with `STRIPE_SECRET_KEY`.

---

## 2. Storage Rules

| Rule | Detail |
|---|---|
| **Never hardcode secrets** | No secret may appear in source code, configuration files committed to source control, or documentation |
| **Environment variables at runtime** | Secrets are injected as environment variables by the deployment platform (e.g., CI/CD secrets, container orchestration secrets) |
| **Secrets manager for production** | Production secrets must be stored in a secrets manager (e.g., AWS Secrets Manager, HashiCorp Vault, GitHub Actions secrets) |
| **No secrets in logs** | Application code must not log any secret value, even partially |
| **No secrets in error responses** | Error messages returned to clients must never include secret values or raw Stripe API keys |
| **Separate per-environment secrets** | Each environment (production, staging, development) uses its own Stripe API keys and webhook secrets |

---

## 3. Environment Segregation

| Environment | Stripe Key Prefix | Webhook Secret | Notes |
|---|---|---|---|
| Production | `sk_live_xxx` | Unique `whsec_xxx` registered in Stripe Dashboard | Live transactions; strictest access |
| Staging | `sk_test_xxx` | Unique `whsec_xxx` registered for staging endpoint | Test mode; no real charges |
| Development / Local | `sk_test_xxx` | Local Stripe CLI webhook secret (`whsec_xxx` from `stripe listen`) | Developer machines; not shared |

**Production keys must never be used in non-production environments.**

---

## 4. Access Control

| Role | `STRIPE_SECRET_KEY` | `STRIPE_WEBHOOK_SECRET` | `STRIPE_PUBLISHABLE_KEY` |
|---|---|---|---|
| Backend service (runtime) | Read via env var | Read via env var | Not needed |
| Frontend application | Never | Never | Bundled/exposed |
| CI/CD pipeline | Write (deploy only) | Write (deploy only) | Write (deploy only) |
| Developer (local) | Read (test key only) | Read (Stripe CLI local) | Read |
| Logs / monitoring | Never | Never | Allowed |

---

## 5. Key Rotation Policy

| Event | Action | Timeline |
|---|---|---|
| Scheduled rotation | Roll `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET` for all environments | Annually |
| Suspected compromise | Immediately revoke and replace the affected key in Stripe Dashboard and all environments | Within 1 hour of detection |
| Engineer offboarding | Rotate any keys the engineer had access to | Same day |
| Security audit | Review access and rotate if any key age exceeds 12 months | Per audit cycle |

### Rotation Steps

1. Generate new key in Stripe Dashboard (old key remains active during transition).
2. Update the secret in the deployment secrets store.
3. Redeploy or hot-reload affected services.
4. Verify new key works (test payment or webhook call).
5. Revoke old key in Stripe Dashboard.
6. Confirm no service errors for 30 minutes post-rotation.

---

## 6. Leak Response

If a secret is suspected to be leaked (e.g., accidentally committed to git, visible in logs):

1. **Immediately revoke** the key in the Stripe Dashboard.
2. **Generate and deploy** a replacement key.
3. **Audit recent Stripe API logs** for unauthorized usage (Stripe Dashboard → Developers → Logs).
4. **Purge** the leaked value from all log stores, PR histories, and issue bodies.
5. **Document** the incident in the incident log with timeline, impact assessment, and corrective actions.

---

## 7. Checklist for New Environments

- [ ] Create a new set of Stripe API keys (never copy from another environment)
- [ ] Register a new webhook endpoint in Stripe Dashboard and save the generated `whsec_xxx`
- [ ] Store all secrets in the environment's secrets manager (not in `.env` files committed to git)
- [ ] Add `.env` and `*.env` to `.gitignore`
- [ ] Verify environment-specific key prefixes (`sk_live_` for production, `sk_test_` for others)
- [ ] Confirm no production key is shared with any non-production system
