# Security

Kanshi's security model, threat assumptions, and responsible disclosure.

## Threat Model

### Assets
- **Event data** – normalized logs of agent activity
- **Policy rules** – decision logic (sensitive but not secret)
- **Audit trail** – immutable record of decisions
- **API tokens** – auth credentials for Kanshi API access

### Attackers
- **Internal:** Malicious employee, compromised service account
- **External:** Network intruder, upstream connector compromise
- **Supply chain:** Compromised dependency, malicious connector plugin

### Trust Assumptions
- Connectors are trusted; Kanshi verifies events but not upstream auth
- Postgres database is protected (network isolation, encryption at rest)
- API tokens are rotated and stored securely by callers
- Kanshi has no outbound execution rights

## Security Principles

### 1. Least Privilege
- Kanshi holds read-only API credentials to sources (via connectors)
- No private keys stored in Kanshi
- Admin APIs require separate auth (e.g., JWT with admin scope)
- Database user has minimal permissions (insert events, append audit logs)

### 2. Immutable Audit Logs
- All events and decisions are append-only
- Deletion requires explicit unsafe migrations (logged, audited)
- Timestamps signed (future: Merkle commitments)
- Audit events include actor, action, timestamp, before/after state

### 3. Signed Events (Future)
- Events can be cryptographically signed by connectors
- Kanshi verifies signatures before policy evaluation
- Chains: Each event references prior event hash (blockchain-style)
- Enables forensic certainty and non-repudiation

### 4. Rate Limiting
- Per-connector: max events/min to prevent flooding
- Per-policy: max evaluations/min to prevent DoS
- Per-API-token: quota to prevent abuse
- Configurable; defaults to reasonable limits

### 5. Authentication & Authorization
- **API Tokens:** Opaque, time-limited (default: 7 days)
- **Role-based access:** `read:events`, `write:policies`, `admin:*`
- **Webhook validation:** HMAC-SHA256 signature on POST body
- **Token rotation:** Issued with expiry; no long-lived tokens for prod

### 6. Network Isolation
- Kanshi API listens on private network (not internet-facing directly)
- Connectors communicate via HTTPS with cert pinning (recommended)
- Database access restricted to Kanshi containers
- Cross-origin requests blocked by default

## Security Checklist

Deploy checklist before production:

- [ ] Database encryption at rest enabled
- [ ] Network policies isolate Kanshi to private subnet
- [ ] TLS 1.3+ for all external APIs
- [ ] Rate limits configured
- [ ] Audit log retention policy set (e.g., 7 years)
- [ ] Backup encryption enabled
- [ ] Log aggregation to immutable sink (e.g., S3 with MFA delete)
- [ ] Secrets rotated: DB password, API tokens
- [ ] Admin token separate from read-only token
- [ ] Monitoring alerts on: failed auth, policy exceptions, DB errors
- [ ] Incident response runbook documented

## Configuration Examples

### Environment Variables (Secure)
```bash
# Database
DATABASE_URL=postgres://user:pass@db.internal/kanshi?sslmode=require

# Admin JWT
ADMIN_SECRET=<rotate quarterly>
ADMIN_TOKEN_EXPIRY=7d

# Rate limits
RATE_LIMIT_EVENTS_PER_MIN=1000
RATE_LIMIT_POLICIES_PER_MIN=100

# Audit
AUDIT_LOG_DESTINATION=s3://kanshi-audit/
AUDIT_LOG_RETENTION_DAYS=2555  # ~7 years
```

### Connector Auth (External)
```yaml
# In separate secrets store (Vault, AWS Secrets Manager, etc.)
connectors:
  github:
    token: "${GITHUB_TOKEN}"  # Loaded from env, not stored in Kanshi
    webhook_secret: "${GH_WEBHOOK_SECRET}"
```

## Attack Scenarios & Mitigations

### Scenario 1: Connector Compromise
**Risk:** Attacker injects malicious events via compromised GitHub token.

**Mitigation:**
- Connector auth stored outside Kanshi
- Events validated against schema (incomplete payloads rejected)
- Rate limiting on per-connector basis
- Audit log shows which connector sent event → forensics

### Scenario 2: Policy Rule Injection
**Risk:** Attacker modifies policy to allow dangerous actions.

**Mitigation:**
- Policy updates require admin auth
- Policy versioning & audit trail
- Policies are code-reviewed before merge
- Dry-run mode available for testing

### Scenario 3: Database Exfiltration
**Risk:** Attacker copies event data via SQL injection.

**Mitigation:**
- Parameterized queries only (ORM/query builder)
- No raw SQL in codebase
- Database user permissions scoped (no `DROP TABLE`)
- Encrypted at rest; encryption keys separate

### Scenario 4: Token Theft
**Risk:** Attacker steals API token from logs/env.

**Mitigation:**
- Tokens hashed in logs (never logged in plaintext)
- Environment variables not printed in errors
- Token expiry (max 7 days)
- Token rotation recommended monthly
- Monitoring for unusual usage patterns

## Monitoring & Alerting

**Critical Alerts (page immediately):**
- Authentication failures (5+ in 1 min)
- Audit log write failure
- Database connection loss
- Unusual policy activation/deactivation

**Warnings (notify, no page):**
- High event ingestion rate (> 10x baseline)
- Policy decision distribution anomaly
- Admin API activity outside business hours

**Metrics to Dashboard:**
- Events ingested (timeline)
- Decisions by type (allow/deny/warn ratio)
- Policy evaluation latency (p50, p99)
- Auth token usage (by role)
- Database query performance

## Responsible Disclosure

Found a vulnerability? 

**Do not open a public issue.** Instead:

1. Email `security@kanshi-os.local` with:
   - Description of the issue
   - Steps to reproduce
   - Suggested fix (optional)
   - Your contact info (for coordination)

2. Allow 48 hours for acknowledgment, 7 days for patch

3. We will:
   - Triage and prioritize
   - Develop fix privately
   - Test fix
   - Release patch
   - Publicly disclose with credits (if desired)

**Thank you for keeping Kanshi secure.**
