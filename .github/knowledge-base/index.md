# Knowledge Base

This directory serves as a local, zero-cost knowledge base for the project.
It acts as a fallback when GitHub Copilot Spaces is not available.

Agents and workflows reference this directory via the `knowledge_base` field in `squad-config.json`.

## Structure

```
knowledge-base/
├── index.md                  # This file — overview and navigation
├── ux/                       # UX guidelines and flow definitions (Phase 1)
│   ├── checkout-flow-diagrams.md
│   ├── field-definitions-and-validation.md
│   ├── ux-copy-guidance.md
│   ├── responsive-behavior.md
│   ├── accessibility-checklist.md
│   └── analytics-events.md
├── security/                 # Security model, PCI, and compliance (Phase 2)
│   ├── security-model.md
│   ├── pci-compliance.md
│   ├── webhook-verification.md
│   ├── secret-handling.md
│   ├── data-classification.md
│   └── fraud-controls.md
└── gateway/                  # API contracts, retry policy, and observability (Phase 3)
    ├── api-contracts.md
    ├── error-envelope.md
    ├── retry-policy.md
    ├── provider-adapter.md
    └── observability-fields.md
```

## Domain Index

### UX (Phase 1 — Checkout UX & Flow Design)
- [`ux/checkout-flow-diagrams.md`](ux/checkout-flow-diagrams.md) — Page sequences and payment method flows
- [`ux/field-definitions-and-validation.md`](ux/field-definitions-and-validation.md) — Input validation rules
- [`ux/ux-copy-guidance.md`](ux/ux-copy-guidance.md) — Error and confirmation messages
- [`ux/responsive-behavior.md`](ux/responsive-behavior.md) — Responsive layout expectations
- [`ux/accessibility-checklist.md`](ux/accessibility-checklist.md) — WCAG compliance checklist
- [`ux/analytics-events.md`](ux/analytics-events.md) — Analytics event catalogue

### Security (Phase 2 — Security & Compliance)
- [`security/security-model.md`](security/security-model.md) — Trust boundaries and threat model
- [`security/pci-compliance.md`](security/pci-compliance.md) — SAQ-A scope and requirements
- [`security/webhook-verification.md`](security/webhook-verification.md) — Stripe webhook signature verification
- [`security/secret-handling.md`](security/secret-handling.md) — API key and secret management
- [`security/data-classification.md`](security/data-classification.md) — Data classification and handling rules
- [`security/fraud-controls.md`](security/fraud-controls.md) — Fraud prevention controls

### Gateway & API Contracts (Phase 3 — Gateway & API Contracts)
- [`gateway/api-contracts.md`](gateway/api-contracts.md) — Endpoint definitions, request/response models, and lifecycle states
- [`gateway/error-envelope.md`](gateway/error-envelope.md) — Structured error codes and client-safe failure semantics
- [`gateway/retry-policy.md`](gateway/retry-policy.md) — Timeout, backoff, idempotency, and compensation logic
- [`gateway/provider-adapter.md`](gateway/provider-adapter.md) — Provider abstraction interface and Stripe adapter contract
- [`gateway/observability-fields.md`](gateway/observability-fields.md) — Correlation fields, log structure, metrics, and trace spans

## Usage

- Templates reference `{{KB_NAME}}`, `{{KB_TYPE}}`, and `{{KB_URL}}`; `{{KB_URL}}` resolves to this directory path (or Copilot Space URL if configured).
- Agents can read files from this directory for context during issue execution.
- Add project-specific documentation here during the discovery/documentation phase and update throughout later phases.
