# Knowledge Base

This directory serves as a local, zero-cost knowledge base for the project.
It acts as a fallback when GitHub Copilot Spaces is not available.

Agents and workflows reference this directory via the `knowledge_base` field in `squad-config.json`.

## Structure

```
knowledge-base/
├── index.md                          # This file — overview and navigation
└── security/                         # Phase 2 — Security & Compliance deliverables
    ├── security-model.md             # Threat boundaries, controls, and trust assumptions
    ├── pci-compliance.md             # PCI DSS scope, SAQ-A assumptions, and controls summary
    ├── data-classification.md        # Sensitive-field handling, masking, and retention rules
    ├── secret-handling.md            # Storage, access control, and rotation policy
    ├── webhook-verification.md       # Stripe webhook signature validation and deduplication
    └── fraud-controls.md             # Abuse prevention, velocity controls, and monitoring
```

## Phase 2 — Security & Compliance

| Document | Description |
|---|---|
| [security-model.md](security/security-model.md) | Trust boundaries, threat model, and security controls for checkout payments |
| [pci-compliance.md](security/pci-compliance.md) | PCI DSS SAQ-A scope definition, tokenization approach, and compliance checklist |
| [data-classification.md](security/data-classification.md) | Field-level classification, masking patterns, and retention rules |
| [secret-handling.md](security/secret-handling.md) | Stripe key storage, environment segregation, and rotation policy |
| [webhook-verification.md](security/webhook-verification.md) | Stripe webhook signature verification and event deduplication policy |
| [fraud-controls.md](security/fraud-controls.md) | Velocity controls, audit logging, and incident response runbook |

## Usage

- Templates reference `{{KB_NAME}}`, `{{KB_TYPE}}`, and `{{KB_URL}}`; `{{KB_URL}}` resolves to this directory path (or Copilot Space URL if configured).
- Agents can read files from this directory for context during issue execution.
- Add project-specific documentation here during the discovery/documentation phase and update throughout later phases.
