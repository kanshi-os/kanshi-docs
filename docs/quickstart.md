# Quickstart

Get Kanshi running locally in minutes.

## Prerequisites

- Node.js 18+
- pnpm (or npm/yarn)
- Local copy of the `kanshi` repository

## Setup

### 1. Install Dependencies

```bash
cd kanshi
pnpm install
```

### 2. Run Development Server

```bash
pnpm dev
```

This starts:
- **Web UI** – http://localhost:3000
- **API** – http://localhost:3001
- **Worker** (if configured) – background job processor

### 3. Verify Health

```bash
curl -s http://localhost:3001/health | jq
```

Expected response:
```json
{
  "status": "ok",
  "version": "0.0.1"
}
```

### 4. Explore the API

View the OpenAPI schema:

```bash
curl -s http://localhost:3001/openapi.json | jq
```

Or open the Swagger UI:
```
http://localhost:3001/api-docs
```

## First API Call

### Get Events (Empty at Start)

```bash
curl -s http://localhost:3001/api/events \
  -H "Authorization: Bearer <token>" | jq
```

### Ingest a Test Event

```bash
curl -X POST http://localhost:3001/api/events \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "source": "webhook",
    "type": "agent.action",
    "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
    "payload": {
      "agent_id": "demo-bot-1",
      "action": "api_call",
      "endpoint": "https://api.github.com/repos/..."
    }
  }' | jq
```

### Check Policy Decision

```bash
curl -s http://localhost:3001/api/decisions?event_id=<id> \
  -H "Authorization: Bearer <token>" | jq
```

## Next Steps

- **[Architecture](architecture.md)** – Understand the module layout
- **[Connectors](connectors.md)** – Set up event sources
- **[Policy](policy.md)** – Write your first policy rules

## Troubleshooting

### Port Already in Use
```bash
lsof -i :3001  # Find process
kill -9 <pid>  # Kill if needed
```

### Dependencies Not Installed
```bash
pnpm install --force
```

### Type Errors
```bash
pnpm type-check
```

## Common Tasks

| Task | Command |
|------|---------|
| Build for production | `pnpm build` |
| Run tests | `pnpm test` |
| Lint code | `pnpm lint` |
| Format code | `pnpm format` |
| Watch mode | `pnpm dev` (automatic) |
