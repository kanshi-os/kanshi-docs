# Architecture

High-level module structure and event flow in Kanshi.

## Module Overview

```
kanshi/
  apps/
    web/          – React UI (event log, policy dashboards, audit view)
    api/          – Express/Node API (event ingest, policy eval, queries)
  packages/
    core/         – Policy engine, event normalization
    db/           – Schema, migrations, audit log storage
    connectors/   – Connector interfaces and built-ins
    types/        – Shared TypeScript types
```

## Event Flow

```
┌──────────────┐
│  Connectors  │  (GitHub, AWS, webhooks, etc.)
│ (read-only)  │
└──────┬───────┘
       │ Events
       ▼
┌──────────────────┐
│  API: /ingest    │  Receive + validate
└──────┬───────────┘
       │ Normalized
       ▼
┌──────────────────┐
│  Policy Engine   │  Evaluate rules
└──────┬───────────┘
       │ Decisions
       ▼
┌──────────────────┐
│  Storage Layer   │  Immutable audit logs
├──────────────────┤  - Events
│  (Postgres)      │  - Decisions
└──────┬───────────┘  - Timestamps
       │
       ▼
    ┌─────────────┐
    │  Web UI     │  Query & visualize
    │ & API       │
    └─────────────┘
```

## Trust Boundaries

### Connector Layer
- **Owns:** Authentication, private key access, rate limiting
- **Provides:** Normalized events only; credentials never passed to Kanshi
- **Trust Model:** Kanshi trusts connector output; doesn't validate upstream auth

### Kanshi Core
- **Owns:** Policy rules, event routing, audit logs
- **Provides:** Read-only visibility; policy decisions
- **Trust Model:** Immutable logs; audit trail is the source of truth

### Runtime/External Systems
- **Owns:** Execution, enforcement decisions, state mutation
- **Interacts with:** Kanshi via event ingest + decision queries
- **Trust Model:** External systems decide to act on Kanshi recommendations

## Key Decisions

**No key custody.** Kanshi never stores or accesses external credentials.
- Rationale: Reduces attack surface; secrets stay at the boundary.

**Events, not commands.** Kanshi ingests events; doesn't send commands.
- Rationale: Decoupled, auditable, easier to reason about.

**Immutable audit trail.** All decisions logged permanently.
- Rationale: Compliance, debugging, forensics.

**Read-only observability.** Kanshi sees; external systems enforce.
- Rationale: Separation of concerns, loose coupling.

## Data Flow Example

1. GitHub connector polls for new actions on a repo
2. Connector sends normalized event to `/api/events`
3. API validates schema, stores event
4. Policy engine evaluates against rules (e.g., "flag if modified files > 50")
5. Decision recorded: `{event_id, decision: "warn", risk_score: 65}`
6. Web UI shows decision in timeline
7. External enforcement system queries `/api/decisions` → decides to review PR vs. auto-merge

## API Entry Points

| Endpoint | Purpose |
|----------|---------|
| `POST /api/events` | Ingest new event |
| `GET /api/events` | Query event log |
| `GET /api/decisions` | Query policy decisions |
| `GET /api/policies` | List active policies |
| `POST /api/policies` | Create/update policy (admin) |
| `GET /health` | Health check |
| `GET /openapi.json` | API schema |

## Storage

**Postgres** (or compatible):
- Events table: id, source, type, payload, timestamp, connector_id
- Decisions table: id, event_id, policy_id, decision (allow/deny/warn), risk_score, timestamp
- Audit table: all mutations with actor, timestamp, change log

All tables are append-only or immutable-log style for compliance.
