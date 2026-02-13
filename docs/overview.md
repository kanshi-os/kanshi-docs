# Overview

**Kanshi is a control plane for AI agents: it ingests events, evaluates policies, and maintains immutable audit trails—without holding keys or executing agents.**

## What Kanshi Is

- **Event ingestion** – Receive events from connectors (GitHub, AWS, webhooks, etc.)
- **Policy evaluation** – Apply rules to decide outcomes (allow/deny/warn + risk scoring)
- **Audit trail** – Immutable log of all events and decisions
- **Read-only visibility** – Observe agent activity without runtime custody
- **Integration boundary** – Connectors normalize payloads; execution stays external

## What Kanshi Is NOT

- **Not an agent runtime** – Agents run elsewhere; Kanshi observes them
- **Not a key manager** – Private keys never enter Kanshi systems
- **Not a policy engine alone** – Policy is part of a larger observability stack
- **Not a message bus** – Events flow in; no outbound command execution from Kanshi core

## Core Concepts

### Control Plane
A centralized system that coordinates policy and observability without managing runtime execution. Kanshi acts as the "eyes and decision-maker" for AI agent fleets.

### Connectors
Adapters that pull/push events and normalize payloads. Each connector owns its auth boundary; Kanshi never sees external credentials.

### Policy Evaluation
Rules engine that takes normalized events and outputs decisions:
- **allow** – proceed
- **deny** – block (if enforced externally)
- **warn** – flag attention (logged, not blocking)
- **risk_score** – numeric severity (0–100)

### Audit Trail
Immutable, tamper-evident log of all events and policy decisions. Every action is recorded with timestamp, actor, and outcome.

### Read-Only Visibility
Kanshi observes agent behavior post-facto. No Kanshi decision can directly halt execution; enforcement is the responsibility of the system calling Kanshi.

## Glossary

| Term | Definition |
|------|-----------|
| **Agent** | Autonomous software system (e.g., LLM-powered bot) |
| **Connector** | Event source/sink adapter |
| **Event** | Normalized structured data (timestamp, type, payload) |
| **Policy** | Rule set that evaluates events → decisions |
| **Decision** | Output of policy evaluation (allow/deny/warn + score) |
| **Audit Trail** | Immutable log of events and decisions |
| **Control Plane** | Centralized observability and policy system (Kanshi) |
| **Runtime** | Where agents actually execute (external to Kanshi) |

## Design Philosophy

**Least privilege.** Kanshi holds read-only data; external systems enforce decisions.

**Transparency.** Every event and decision is logged and queryable.

**Trust boundaries.** Auth, keys, and execution live outside Kanshi.

**Simplicity.** One job: observe, evaluate, audit.
