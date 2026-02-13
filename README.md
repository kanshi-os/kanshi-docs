# kanshi-docs

**Control plane for AI agents — event ingestion, policy evaluation, and audit trails without runtime custody.**

Documentation and architecture guide for Kanshi OS.

## Quick Navigation

- **[Overview](docs/overview.md)** – What Kanshi is, core concepts, glossary
- **[Quickstart](docs/quickstart.md)** – Local setup and first API calls
- **[Architecture](docs/architecture.md)** – Module layout, event flow, trust boundaries
- **[Connectors](docs/connectors.md)** – Event sources, payload normalization, auth boundaries
- **[Policy](docs/policy.md)** – Policy model, evaluation, decision types
- **[Security](docs/security.md)** – Threat model, principles, responsible disclosure

## Documentation Structure

```
kanshi-docs/
  docs/          – User-facing documentation
  adr/           – Architecture Decision Records
  diagrams/      – Diagrams and visual assets
```

## Conventions

**Writing Style**
- Concise, dev-native tone. Avoid marketing language.
- Use bullet points for clarity; full sentences for explanations.
- Consistent terminology: "control plane", "observability", "policy evaluation", "audit trail", "connectors".

**File Format**
- Markdown (.md) for all docs.
- Keep files focused: one concept per file.
- Link liberally between docs.

**ADR Usage**
- Use [Architecture Decision Records](adr/) to document significant decisions.
- Template: [adr/0000-template.md](adr/0000-template.md)
- ADRs are immutable once merged; superseded records reference their successors.

## Start Here

New to Kanshi? Read [Overview](docs/overview.md) first, then [Quickstart](docs/quickstart.md).

**Contributing docs?** Add a new `.md` file to `docs/` and link it from the Quick Navigation section above. Create ADRs for architectural decisions.