# Payment Checkout Template Profile

This profile defines a complete 8-phase workflow for building and releasing a robust payment checkout capability in the HIS framework.

## What this profile is for

Use this profile when your initiative requires end-to-end checkout delivery, from requirements discovery through launch readiness, with strong coverage for security, payment reliability, and operational visibility.

## Phases at a glance

1. **Phase 0**: Discovery & Requirements
2. **Phase 1**: Checkout UX & Flow Design
3. **Phase 2**: Security & Compliance
4. **Phase 3**: Gateway & API Contracts
5. **Phase 4**: Backend Payment Orchestration
6. **Phase 5**: Frontend Checkout Implementation
7. **Phase 6**: Testing & Payment Simulation
8. **Phase 7**: Observability & Release Readiness

## How to activate

Update your `squad-config.json`:

- Set `template_profile` to `checkout`
- Set `labels.trigger` to your checkout initiative trigger label

Example:

```json
{
  "template_profile": "checkout",
  "labels": {
    "trigger": "checkout"
  }
}
```

## Template variables

This profile uses the following variables in phase templates:

- `{{ISSUE_NUMBER}}`
- `{{PAYMENT_PROVIDER}}`
- `{{SUPPORTED_PAYMENT_METHODS}}`
- `{{DEFAULT_CURRENCY}}`
- `{{CHECKOUT_MODE}}`
- `{{PCI_SCOPE}}`
- `{{FRAUD_PROVIDER}}`

## Worked simulation

See [reference-simulation.md](reference-simulation.md) for a worked example of initiative creation, phase fan-out behavior, sample outputs, edge-case handling, and dependency timeline.
