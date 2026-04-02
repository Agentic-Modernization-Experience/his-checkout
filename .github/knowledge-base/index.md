# Knowledge Base

This directory serves as a local, zero-cost knowledge base for the project.
It acts as a fallback when GitHub Copilot Spaces is not available.

Agents and workflows reference this directory via the `knowledge_base` field in `squad-config.json`.

## Structure

```
knowledge-base/
├── index.md                  # This file — overview and navigation
├── ux/                       # Phase 1 — UX & Flow Design
│   ├── checkout-flow-diagrams.md
│   ├── accessibility-checklist.md
│   ├── analytics-events.md
│   ├── field-definitions-and-validation.md
│   ├── responsive-behavior.md
│   └── ux-copy-guidance.md
├── security/                 # Phase 2 — Security & Compliance
│   ├── pci-compliance.md
│   ├── security-model.md
│   ├── data-classification.md
│   ├── secret-handling.md
│   ├── webhook-verification.md
│   └── fraud-controls.md
└── frontend/                 # Phase 5 — Frontend Checkout Implementation
    ├── component-architecture.md
    ├── stripe-sdk-integration.md
    ├── error-handling-strategy.md
    ├── analytics-implementation.md
    └── accessibility-compliance.md
```

## Phase 1 — UX & Flow Design

| Document | Description |
|----------|-------------|
| [`ux/checkout-flow-diagrams.md`](ux/checkout-flow-diagrams.md) | Full checkout flow diagrams: guest, returning user, credit card, Pix, Boleto, and recovery paths |
| [`ux/accessibility-checklist.md`](ux/accessibility-checklist.md) | WCAG 2.1 AA checklist for keyboard navigation, screen readers, contrast, and form errors |
| [`ux/analytics-events.md`](ux/analytics-events.md) | Analytics event schemas, mandatory base properties, and SAQ-A compliance verification |
| [`ux/field-definitions-and-validation.md`](ux/field-definitions-and-validation.md) | Field definitions, validation rules, and submission states for all checkout steps |
| [`ux/responsive-behavior.md`](ux/responsive-behavior.md) | Breakpoints, layout behavior, and component-level responsive rules |
| [`ux/ux-copy-guidance.md`](ux/ux-copy-guidance.md) | User-facing copy guidelines, tone, error messages, and CTA labels |

## Phase 2 — Security & Compliance

| Document | Description |
|----------|-------------|
| [`security/pci-compliance.md`](security/pci-compliance.md) | SAQ-A scope definition, cardholder data boundaries, and annual self-assessment checklist |
| [`security/security-model.md`](security/security-model.md) | Overall security model: tokenization, secrets, transport, and access controls |
| [`security/data-classification.md`](security/data-classification.md) | Data classification tiers and handling rules for checkout data |
| [`security/secret-handling.md`](security/secret-handling.md) | Secret management: Stripe keys, webhook secrets, rotation policies |
| [`security/webhook-verification.md`](security/webhook-verification.md) | Stripe webhook signature verification and deduplication approach |
| [`security/fraud-controls.md`](security/fraud-controls.md) | Fraud controls, Stripe Radar configuration, and manual review triggers |

## Phase 5 — Frontend Checkout Implementation

| Document | Description |
|----------|-------------|
| [`frontend/component-architecture.md`](frontend/component-architecture.md) | Component tree, state ownership, step transitions, and reload recovery design |
| [`frontend/stripe-sdk-integration.md`](frontend/stripe-sdk-integration.md) | Stripe.js initialization, Elements setup, payment confirmation flows, and 3DS handling |
| [`frontend/error-handling-strategy.md`](frontend/error-handling-strategy.md) | Client-safe error mapping for field errors, declines, 4xx, and 5xx failures |
| [`frontend/analytics-implementation.md`](frontend/analytics-implementation.md) | Analytics layer architecture, per-component firing points, and SAQ-A privacy enforcement |
| [`frontend/accessibility-compliance.md`](frontend/accessibility-compliance.md) | Component-level WCAG 2.1 AA implementation, keyboard flow verification, and validation matrix |

## Usage

- Templates reference `{{KB_NAME}}`, `{{KB_TYPE}}`, and `{{KB_URL}}`; `{{KB_URL}}` resolves to this directory path (or Copilot Space URL if configured).
- Agents can read files from this directory for context during issue execution.
- Add project-specific documentation here during the discovery/documentation phase and update throughout later phases.
