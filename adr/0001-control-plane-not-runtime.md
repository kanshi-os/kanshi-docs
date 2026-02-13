# ADR 0001: Control Plane, Not Runtime

**Status:** Accepted

**Date:** 2025-02-13

**Author(s):** Kanshi Core Team

---

## Context

Kanshi's scope needed clear definition early to guide architecture and product decisions. The team evaluated whether Kanshi should:

1. **Include an agent runtime** – host and execute agents directly
2. **Be a control plane only** – observe and policy-manage externally-running agents
3. **Be a hybrid** – optional runtime with control features

This decision affects:
- Key management & trust boundaries
- Deployment model & operational burden
- Integration complexity with existing agent platforms
- Regulatory/compliance posture (key custody, audit)

## Decision

**Kanshi is a control plane, not an agent runtime.**

Kanshi's responsibility is:
- **Ingest** events from agent systems
- **Evaluate** policies and produce decisions
- **Audit** all activity immutably
- **NOT** host, execute, or custody agents

Agents run in external systems (user infrastructure, LLM platforms, etc.); Kanshi observes and guides via policy.

## Rationale

### Why Not Include a Runtime?

**Key Custody Problem**
- If Kanshi runs agents, Kanshi must store/access API keys, credentials, signing keys
- Single-point-of-failure for security
- Complicates key rotation, audit, compliance
- Increases blast radius of Kanshi compromise

**Operational Complexity**
- Runtime adds: scheduling, resource management, fault recovery, auto-scaling
- Diverts focus from core control-plane features
- Multiplies deployment models (Docker, K8s, Lambda, etc.)
- Each deployment model has unique security concerns

**Limited Value**
- Most users already have infrastructure to run agents
- Tightly coupling control + runtime limits flexibility
- Third-party runtimes (Anthropic, OpenAI, Hugging Face) are better optimized

### Why Control Plane Only?

**Clean Separation**
- Kanshi: observability + policy (read-heavy, append-only logs)
- External: execution + state mutation (user's responsibility)
- Each system does one thing well

**No Key Access Required**
- Connectors own authentication; Kanshi sees only events
- Credentials never enter Kanshi
- Easier to audit; smaller attack surface
- Standards-compliant (no privileged key storage)

**Flexible Integration**
- Works with any agent platform (LLM APIs, local executors, cloud services)
- Agents can enforce Kanshi decisions however they choose
- Decoupled: updates to Kanshi don't require agent redeployment

**Deployment Simplicity**
- Single deployment model: stateless API + Postgres
- Scales horizontally by design
- No resource contention with agent workloads
- Easier to reason about failure modes

## Consequences

### Positive
- ✓ Minimal key/secret management
- ✓ Clear trust boundaries (agent runtime is untrusted; Kanshi is authoritative)
- ✓ Simpler architecture & faster time-to-value
- ✓ Compatible with multi-platform agent deployments
- ✓ Reduced operational burden and on-call load

### Negative
- ✗ Agents must implement their own execution engines
- ✗ Kanshi decisions are advisory unless agent runtime enforces them
- ✗ Requires discipline from integrators to respect Kanshi "deny" decisions
- ✗ Some users may expect turnkey agent hosting

### Neutral
- ~ Connectors become critical boundary components (auth, normalization)
- ~ Integration requires agent instrumentation (event emission)
- ~ Enforcement depends on external system compliance

## Implementation Plan

1. **Define Connector Interface** (docs/connectors.md)
   - Specify event schema, auth model, normalization

2. **Architect Policy Engine** (docs/architecture.md)
   - Evaluation rules, decision output format
   - No execution primitives

3. **Build API** (not custody-related)
   - `/api/events` - ingest only
   - `/api/decisions` - query only
   - `/api/policies` - CRUD for rules (no execution)

4. **Document Integration Pattern**
   - Show how external agents call Kanshi
   - Examples of decision enforcement

5. **No Additions Later**
   - If feature request asks for agent hosting, defer to ADR 0002 (or decline)

## References

- [docs/architecture.md](../docs/architecture.md#trust-boundaries) – Trust boundaries diagram
- [docs/security.md](../docs/security.md) – Security model assumes no key custody
- [docs/connectors.md](../docs/connectors.md) – Connector auth boundary
- Related: Potential future ADR on "Multi-Tenant Isolation" or "Agent Fingerprinting"

---

## Decision History

- **2025-02-13:** Accepted. Aligns with operator feedback and security best practices.
