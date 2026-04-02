# Knowledge Base

This directory serves as a local, zero-cost knowledge base for the project.
It acts as a fallback when GitHub Copilot Spaces is not available.

Agents and workflows reference this directory via the `knowledge_base` field in `squad-config.json`.

## Structure

```
knowledge-base/
├── index.md                  # This file — overview and navigation
├── ux/                       # Phase 1: UX & Flow Design deliverables
│   ├── checkout-flow-diagrams.md
│   ├── field-definitions-and-validation.md
│   ├── ux-copy-guidance.md
│   ├── accessibility-checklist.md
│   ├── responsive-behavior.md
│   └── analytics-events.md
├── security/                 # Phase 2: Security & Compliance (forthcoming)
├── architecture/             # System architecture and design decisions
├── business-rules/           # Business logic documentation
├── decisions/                # Key decisions, trade-offs, and rationale
└── patterns/                 # Coding patterns, conventions, best practices
```

## Usage

- Templates reference `{{KB_NAME}}`, `{{KB_TYPE}}`, and `{{KB_URL}}`; `{{KB_URL}}` resolves to this directory path (or Copilot Space URL if configured).
- Agents can read files from this directory for context during issue execution.
- Add project-specific documentation here during the discovery/documentation phase and update throughout later phases.

---

## Phase 1 — Checkout UX & Flow Design

> Profile: Hosted Checkout | Payment Methods: credit_card, pix, boleto | Provider: Stripe | Currency: BRL

| Document | Description |
|----------|-------------|
| [checkout-flow-diagrams.md](ux/checkout-flow-diagrams.md) | Primary and alternate checkout paths; guest vs. returning user branching; per-method step transitions; recovery and fallback flows; modal inventory |
| [field-definitions-and-validation.md](ux/field-definitions-and-validation.md) | Complete field inventory for all steps; validation rules and timing; submission states; idempotency and retry rules |
| [ux-copy-guidance.md](ux/ux-copy-guidance.md) | All user-facing copy: CTAs, field errors, decline messages, system errors, Pix/Boleto-specific messages, confirmation copy (pt-BR + en reference) |
| [accessibility-checklist.md](ux/accessibility-checklist.md) | WCAG 2.1 AA conformance checklist; keyboard flow; focus management; screen-reader expectations; ARIA live regions; Stripe Elements accessibility |
| [responsive-behavior.md](ux/responsive-behavior.md) | Mobile and desktop layout behavior; breakpoint definitions; component-level responsive rules; typography scale |
| [analytics-events.md](ux/analytics-events.md) | Analytics event schemas for all checkout moments; mandatory base properties; SAQ-A compliance verification checklist |
