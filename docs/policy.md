# Policy

Kanshi's policy engine evaluates events and produces decisions.

## Policy Model

A **policy** is a set of **rules** that evaluate **events** and produce **decisions**.

### Decision Types

| Type | Meaning | Use Case |
|------|---------|----------|
| `allow` | Proceed without concern | Known-safe patterns |
| `deny` | Block or escalate | High-risk patterns |
| `warn` | Flag for attention | Suspicious but allowed | 
| `info` | Informational | Logging and metrics |

### Risk Scores

Each decision includes an optional **risk_score** (0–100):
- **0–20:** Low risk (routine events)
- **21–50:** Medium risk (review recommended)
- **51–80:** High risk (escalate)
- **81–100:** Critical (immediate action)

## Example Policy Rule (Pseudo-DSL)

```
rule: "high_code_change_rate"
  description: "Warn if agent commits > 50 files in < 1 hour"
  
  when:
    event.type = "git.push"
    AND event.payload.files_changed > 50
    AND event.payload.duration_minutes < 60
  
  then:
    decision: "warn"
    risk_score: 70
    message: "High-velocity code change from {agent_id}"
    tags: ["code_quality", "velocity"]

---

rule: "external_api_quota_exceeded"
  description: "Deny if agent exceeds API quota"
  
  when:
    event.type = "api.rate_limit"
    AND event.payload.remaining <= 0
  
  then:
    decision: "deny"
    risk_score: 85
    message: "API quota exhausted for {service}"
    tags: ["api", "quota"]
```

## Example Policy (JSON)

```json
{
  "id": "policy-01",
  "name": "Agent Safety Baseline",
  "version": "1.0.0",
  "enabled": true,
  "rules": [
    {
      "id": "rule-01",
      "name": "Flag large transactions",
      "condition": {
        "type": "event.type",
        "op": "eq",
        "value": "transaction.created"
      },
      "decision": "warn",
      "risk_score": 60,
      "metadata": {
        "tags": ["finance", "transaction"]
      }
    },
    {
      "id": "rule-02",
      "name": "Block unauthorized deploys",
      "condition": {
        "type": "and",
        "conditions": [
          {
            "type": "event.payload.environment",
            "op": "eq",
            "value": "production"
          },
          {
            "type": "event.payload.approved",
            "op": "eq",
            "value": false
          }
        ]
      },
      "decision": "deny",
      "risk_score": 95
    }
  ]
}
```

## Policy Evaluation Flow

```
1. Event arrives at /api/events
   ↓
2. Schema validation (passes or rejects)
   ↓
3. Policy engine loads active policies
   ↓
4. For each policy:
     - Evaluate all rules
     - Aggregate decisions (first match, highest risk, etc.)
   ↓
5. Store decision record
   ↓
6. Return decision(s) to caller
   ↓
7. External system decides action (enforce, log, escalate)
```

## Decision Output

```json
{
  "event_id": "evt-123",
  "timestamp": "2025-02-13T10:30:00Z",
  "policy_id": "policy-01",
  "rule_id": "rule-02",
  "decision": "deny",
  "risk_score": 95,
  "message": "Block unauthorized deploys",
  "rationale": "Deployment to production without approval",
  "metadata": {
    "tags": ["deploy", "security"],
    "agent_id": "ci-bot-1",
    "source": "github"
  }
}
```

## UI Display of Decisions

### Timeline View
```
10:30 | evt-123 | GitHub | workflow triggered    | WARN (70)
10:31 | evt-124 | GitHub | 62 files changed       | WARN (75)
10:32 | evt-125 | GitHub | PR created to prod     | DENY (95)
```

### Decision Summary
```
Policy: Agent Safety Baseline
Active Rules: 15
Total Decisions (24h): 127
  ✓ Allow: 95
  ⚠ Warn: 28
  ✗ Deny: 4
Avg Risk Score: 35
```

### Audit Trail
```
Timestamp      | Event ID | Type      | Decision | Risk | Agent    | Source
2025-02-13 10:30 | evt-123  | api_call | allow    | 15   | bot-1    | github
2025-02-13 10:31 | evt-124  | git.push | warn     | 70   | bot-1    | github
2025-02-13 10:32 | evt-125  | deploy   | deny     | 95   | bot-1    | webhook
```

## Policy Management

### Create/Update Policy
```bash
curl -X POST http://localhost:3001/api/policies \
  -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d @policy.json
```

### List Active Policies
```bash
curl http://localhost:3001/api/policies \
  -H "Authorization: Bearer <token>"
```

### Disable/Delete Policy
```bash
curl -X DELETE http://localhost:3001/api/policies/policy-01 \
  -H "Authorization: Bearer <admin-token>"
```

## Best Practices

- **Keep rules simple** – one concern per rule
- **Use meaningful risk scores** – calibrate to your threat model
- **Version policies** – track changes; allow rollback
- **Test policies** – dry-run on historical events before enabling
- **Monitor decision distribution** – alert on sudden changes
- **Document rationale** – explain why each rule exists
