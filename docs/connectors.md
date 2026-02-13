# Connectors

Connectors are the boundary between Kanshi and external systems. They normalize events and maintain auth boundaries.

## Connector Responsibilities

### Event Polling/Pushing
- Pull events from source systems (GitHub webhooks, AWS EventBridge, etc.)
- Push normalized events to Kanshi `/api/events`
- Handle source-specific auth (API keys, OAuth, etc.)

### Payload Normalization
Transform source events into Kanshi's standard format:
```json
{
  "source": "github",
  "type": "pull_request.opened",
  "timestamp": "2025-02-13T10:30:00Z",
  "connector_id": "github-main",
  "payload": {
    "agent_id": "bot-pr-reviewer",
    "action": "created_pr",
    "details": { ... }
  }
}
```

### Auth Boundaries
- **Connector owns credentials** (API keys, tokens, signing secrets)
- **Kanshi never sees them** – stored outside Kanshi or in connector config only
- **Events are public** – payload contains no secrets
- **Audit logged** – which connector ingested the event, timestamp, outcome

## Connector Interface

```typescript
interface Connector {
  id: string;
  name: string;
  
  // Initialize with config (creds, webhooks, etc.)
  init(config: ConnectorConfig): Promise<void>;
  
  // Poll or receive event
  fetch(): Promise<NormalizedEvent[]>;
  
  // Validate webhook signature (if applicable)
  validateSignature(payload: Buffer, signature: string): boolean;
  
  // Health check
  health(): Promise<boolean>;
}

interface NormalizedEvent {
  source: string;
  type: string;
  timestamp: string;  // ISO 8601
  connector_id: string;
  payload: Record<string, any>;
}
```

## Example Connector Config (YAML)

```yaml
connectors:
  github:
    enabled: true
    type: github
    config:
      owner: "my-org"
      repo: "my-repo"
      token: "${GITHUB_TOKEN}"  # From env
      events:
        - pull_request
        - push
        - issues
      webhook_url: "https://kanshi.internal/webhooks/github"
      webhook_secret: "${WEBHOOK_SECRET}"
  
  aws:
    enabled: true
    type: aws
    config:
      region: "us-east-1"
      eventbridge_rule_name: "kanshi-ingestion"
      role_arn: "arn:aws:iam::123456789:role/kanshi-reader"
      events:
        - ec2.instance.*
        - iam.user.*
```

## Placeholder Connectors

These are defined but not fully implemented; use as reference:

| Connector | Source | Event Types |
|-----------|--------|-------------|
| **OpenClaw** | LLM function calls | `llm.function_call`, `token.usage` |
| **Moltbook** | Notebook executions | `notebook.cell_executed`, `model.trained` |
| **4claw** | Code changes | `git.push`, `review.started` |
| **Webhook** | Generic HTTP POST | Custom payloads (must match schema) |
| **GitHub** | Repository events | `pull_request.*`, `push`, `issues.*` |
| **AWS** | EventBridge events | `ec2.*`, `iam.*`, `s3.*` |

## Building a Custom Connector

1. **Implement `Connector` interface** – `id`, `init()`, `fetch()`, `health()`
2. **Normalize payloads** – map source events to Kanshi schema
3. **Store credentials externally** – never pass to Kanshi
4. **Register in config** – add to `connectors.yaml` or env vars
5. **Test normalization** – verify schema compliance with `/api/events`

### Minimal Example

```typescript
export class WebhookConnector implements Connector {
  id = "webhook-custom";
  name = "Custom Webhook";
  
  private secret: string;
  
  async init(config: ConnectorConfig) {
    this.secret = config.webhook_secret;
  }
  
  async fetch(): Promise<NormalizedEvent[]> {
    // For webhooks, events arrive via HTTP POST to /webhooks/:id
    // This method is optional; implement for polling-style sources
    return [];
  }
  
  validateSignature(payload: Buffer, signature: string): boolean {
    const hmac = crypto.createHmac('sha256', this.secret);
    const expected = hmac.update(payload).digest('hex');
    return crypto.timingSafeEqual(expected, signature);
  }
  
  async health(): Promise<boolean> {
    return true; // Passive connector; always ready
  }
}
```

## Security Notes

- **Credentials in config only** – use env var interpolation (e.g., `${GITHUB_TOKEN}`)
- **Webhook validation** – always verify signatures (HMAC-SHA256, etc.)
- **Rate limiting** – implement at connector level, not Kanshi core
- **Error handling** – log failures without exposing secrets
- **Least privilege** – request only necessary scopes/permissions from sources
