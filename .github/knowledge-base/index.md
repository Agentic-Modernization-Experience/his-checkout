# Knowledge Base

This directory serves as a local, zero-cost knowledge base for the project.
It acts as a fallback when GitHub Copilot Spaces is not available.

Agents and workflows reference this directory via the `knowledge_base` field in `squad-config.json`.

## Structure

```
knowledge-base/
├── index.md                        # This file — overview and navigation
├── ux/                             # Phase 1 — Checkout UX & Flow Design
│   ├── checkout-flow-diagrams.md
│   ├── field-definitions-and-validation.md
│   ├── ux-copy-guidance.md
│   ├── responsive-behavior.md
│   ├── accessibility-checklist.md
│   └── analytics-events.md
├── security/                       # Phase 2 — Security & Compliance
│   ├── security-model.md
│   ├── pci-compliance.md
│   ├── webhook-verification.md
│   ├── data-classification.md
│   ├── fraud-controls.md
│   └── secret-handling.md
└── payment/                        # Phase 4 — Backend Payment Orchestration
    ├── state-machine.md
    ├── service-layer.md
    ├── persistence-model.md
    ├── webhook-handler.md
    ├── compensation-refund.md
    └── reliability.md
```

## Usage

- Templates reference `{{KB_NAME}}`, `{{KB_TYPE}}`, and `{{KB_URL}}`; `{{KB_URL}}` resolves to this directory path (or Copilot Space URL if configured).
- Agents can read files from this directory for context during issue execution.
- Add project-specific documentation here during the discovery/documentation phase and update throughout later phases.

---

## Phase 4 — Backend Payment Orchestration

| File | Contents |
|---|---|
| `payment/state-machine.md` | Order and payment attempt state machines, transition tables, invariants, MAX_ATTEMPTS policy |
| `payment/service-layer.md` | Orchestration service operations (`createOrder`, `initiatePayment`, `confirmPayment`, `cancelOrder`, `initiateRefund`), idempotency key design, error contract |
| `payment/persistence-model.md` | Schema for `orders`, `payment_attempts`, `refund_records`, `idempotency_keys`, `stripe_processed_events`; atomicity requirements; retention schedule |
| `payment/webhook-handler.md` | Ingestion pipeline, signature verification, deduplication, event dispatch handlers, out-of-order event handling, stale reconciliation job |
| `payment/compensation-refund.md` | Cancellation flows (pre/post capture), full and partial refund operations, inventory reservation/release timing, recovery hooks, audit log format |
| `payment/reliability.md` | Fallback behavior during provider unavailability, retry controls, dead-letter handling, payment-domain logging and tracing fields |

---

## Phase 2 — Security & Compliance

| File | Contents |
|---|---|
| `security/security-model.md` | Trust boundaries, threat model, security controls, data flow |
| `security/pci-compliance.md` | SAQ-A scope, cardholder data boundaries, tokenization approach |
| `security/webhook-verification.md` | Signature verification algorithm, deduplication, accepted events |
| `security/data-classification.md` | Field classification matrix, masking rules, retention periods |
| `security/fraud-controls.md` | Fraud detection signals and controls |
| `security/secret-handling.md` | Secret management requirements |

---

## Phase 1 — Checkout UX & Flow Design

| File | Contents |
|---|---|
| `ux/checkout-flow-diagrams.md` | User journey diagrams for checkout flows |
| `ux/field-definitions-and-validation.md` | Form field definitions and validation rules |
| `ux/ux-copy-guidance.md` | Copy guidelines for checkout UI |
| `ux/responsive-behavior.md` | Responsive design requirements |
| `ux/accessibility-checklist.md` | Accessibility requirements and checklist |
| `ux/analytics-events.md` | Analytics event tracking specification |
