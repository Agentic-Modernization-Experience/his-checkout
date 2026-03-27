# Knowledge Base

This directory serves as a local, zero-cost knowledge base for the project.
It acts as a fallback when GitHub Copilot Spaces is not available.

Agents and workflows reference this directory via the `knowledge_base` field in `squad-config.json`.

## Structure

Organize documentation by domain area. Suggested structure:

```
knowledge-base/
├── index.md                  # This file — overview and navigation
├── architecture/             # System architecture and design decisions
├── business-rules/           # Business logic documentation
├── decisions/                # Key decisions, trade-offs, and rationale
└── patterns/                 # Coding patterns, conventions, best practices
```

## Usage

- Templates reference `{{KB_NAME}}`, `{{KB_TYPE}}`, and `{{KB_URL}}`; `{{KB_URL}}` resolves to this directory path (or Copilot Space URL if configured).
- Agents can read files from this directory for context during issue execution.
- Add project-specific documentation here during the discovery/documentation phase and update throughout later phases.
