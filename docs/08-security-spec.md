# Security Specification: PolyBot Platform

## Threat Model

### Assets

| Asset | Sensitivity | Impact if Compromised |
|-------|------------|----------------------|
| EOA private keys (3 wallets) | **CRITICAL** | Full loss of all funds in compromised wallet |
| Polymarket L2 API keys (key/secret/passphrase) | **HIGH** | Unauthorized order placement, position manipulation |
| USDC collateral (trading capital) | **HIGH** | Financial loss |
| Trading strategy source code | **MEDIUM** | Competitive advantage lost |
| Historical trade data | **LOW** | Privacy concern, strategy reverse-engineering |
| Dashboard credentials | **MEDIUM** | Unauthorized bot control, emergency stop abuse |

### Threat Actors

| Actor | Motivation | Capability | Likelihood |
|-------|-----------|------------|------------|
| **External attacker** | Steal funds via key theft | Medium (VPS exploitation, phishing) | Medium |
| **Polymarket operator** | N/A (trusted, but not custodial) | High (controls order matching) | Low |
| **UMA oracle manipulator** | Profit from false market resolution | Medium-High (documented attacks) | Medium |
| **Competing bot operators** | Front-run strategies, grief | Medium (public mempool, API monitoring) | Medium |
| **VPS provider insider** | Access to disk/memory | Low-Medium (physical access to host) | Low |

### Attack Surface

| Entry Point | Exposure | Mitigation |
|-------------|----------|------------|
| SSH access to VPS | Internet-facing | Key-only auth, Fail2Ban, non-standard port |
| Dashboard web interface (port 8080) | Internet-facing (if exposed) | API key auth, HTTPS, IP allowlisting optional |
| Grafana (port 3000) | Internet-facing (if exposed) | Auth required, read-only default |
| Docker API | Host-only | Not exposed to network (default) |
| .env file on disk | Local filesystem | File permissions 600, encrypted at rest with sops/age |
| Redis (port 6379) | Docker internal only | Not exposed to host; no auth needed (internal network) |
| PostgreSQL (port 5432) | Docker internal only | Not exposed to host; password auth |
| Polymarket API traffic | Outbound HTTPS | TLS 1.2+; HMAC-signed requests |

### STRIDE Analysis

| Threat | Category | Likelihood | Impact | Mitigation |
|--------|----------|-----------|--------|------------|
| Attacker steals private keys from .env | **Spoofing** | Medium | Critical | Encrypt .env with sops/age; Docker secrets for production; file permissions 600; never commit to git |
| Attacker modifies bot config to drain funds | **Tampering** | Low | High | Dashboard auth required; config changes logged in audit table; file integrity monitoring |
| Trades made without audit trail | **Repudiation** | Low | Medium | All orders logged in PostgreSQL audit_log with timestamps; immutable append-only |
| Private keys leaked in logs | **Information Disclosure** | Medium | Critical | structlog configured to redact sensitive fields; grep CI check for key patterns |
| Competing bot reads our order flow | **Information Disclosure** | Low | Medium | Orders submitted over HTTPS; no public mempool (off-chain matching) |
| Rate limit exhaustion prevents trading | **Denial of Service** | Medium | Medium | Client-side rate limiter prevents hitting Polymarket limits; priority queue for critical orders |
| Malicious bot module loaded | **Elevation of Privilege** | Low | High | Bot modules loaded from local filesystem only; no remote code execution; validate ABC compliance |

---

## Authentication & Authorization

### Polymarket API Authentication

| Level | Mechanism | Usage | Storage |
|-------|-----------|-------|---------|
| **L1** (wallet) | EIP-712 signed message with EOA private key | One-time: derive API credentials | Private key in .env (encrypted) |
| **L2** (API) | HMAC-SHA256 with secret; passphrase in header | Every authenticated request | In-memory cache (never written to disk after derivation) |

**Key Derivation Flow**:
1. Load EOA private key from encrypted .env
2. Call `create_or_derive_api_creds()` (idempotent — same key always returns same credentials)
3. Cache `{apiKey, secret, passphrase}` in Wallet Manager memory
4. Secret is used for HMAC computation, never sent with requests
5. Passphrase sent in `POLY_PASSPHRASE` header with every request

### Dashboard Authentication

**MVP**: API key authentication
- Single API key stored in .env (`DASHBOARD_API_KEY`)
- Sent as `Authorization: Bearer <key>` header
- All dashboard API endpoints require valid key
- Emergency Stop endpoint requires key (no unauthenticated access to controls)

**Phase 2+**: Session-based authentication
- Username/password with bcrypt hashing
- JWT session tokens with 24-hour expiry
- CSRF protection for state-changing operations

### Authorization Model

| Role | Permissions | Phase |
|------|------------|-------|
| **Operator** (single user) | Full access to all endpoints and bot controls | Phase 1 |
| **Viewer** (read-only) | Dashboard viewing, no bot control or config changes | Phase 2 |
| **System** (internal services) | Inter-service communication via Redis (no auth needed on internal Docker network) | Phase 1 |

---

## Data Protection

### Encryption at Rest

| Data | Encryption | Mechanism |
|------|-----------|-----------|
| EOA private keys | AES-256 | sops/age encrypted .env file; decrypted at runtime |
| L2 API credentials | Not stored on disk | Derived at startup, held in memory only |
| PostgreSQL data | Filesystem-level | Docker volume; optional LUKS full-disk encryption on VPS |
| Redis data | Not encrypted (internal) | Docker internal network only; AOF on encrypted volume |
| Backup files | AES-256 | gpg encryption before offsite transfer |

### Encryption in Transit

| Connection | Protocol | Verification |
|------------|----------|-------------|
| PolyBot → Polymarket CLOB | HTTPS (TLS 1.2+) | Certificate validation enabled |
| PolyBot → Polymarket WebSocket | WSS (TLS 1.2+) | Certificate validation enabled |
| PolyBot → Polygon RPC | HTTPS (TLS 1.2+) | Certificate validation enabled |
| Browser → Dashboard | HTTPS (recommended) | Let's Encrypt certificate via Caddy/nginx reverse proxy |
| Inter-service (Docker) | Unencrypted (Docker internal network) | Acceptable — not exposed to host or internet |

### PII & Sensitive Data Handling

| Data Type | Classification | Handling |
|-----------|---------------|----------|
| Private keys | **SECRET** | Never logged, never in error messages, never in git |
| API credentials | **SECRET** | In-memory only, never persisted to disk after derivation |
| Wallet addresses | **INTERNAL** | Logged for debugging; not PII (public on blockchain) |
| Trade history | **INTERNAL** | Stored in PostgreSQL; retained indefinitely for audit |
| Dashboard API key | **SECRET** | In .env file; never logged |

### Log Redaction

structlog configured with a processor that redacts patterns matching:
- Private keys (hex strings of 64 characters)
- API keys (UUID format)
- API secrets
- Passphrases
- Bearer tokens

```python
# src/shared/logging_config.py
REDACT_PATTERNS = [
    r'[0-9a-fA-F]{64}',           # Private keys
    r'secret["\s:=]+\S+',          # Secret values
    r'passphrase["\s:=]+\S+',      # Passphrases
    r'Bearer\s+\S+',               # Auth tokens
]
```

---

## API Security

### Rate Limiting (Outbound)

| API | Limit | Enforcement |
|-----|-------|-------------|
| Polymarket CLOB (orders) | 3,500 per 10s per wallet | Client-side token bucket in `rate_limiter.py` |
| Polymarket CLOB (public) | 100 per minute | Client-side counter |
| Gamma API | 60 per minute (estimated) | Client-side counter with backoff |

### Input Validation (Dashboard API)

- All incoming data validated by Pydantic models before processing
- Bot config YAML validated against schema on upload
- Risk parameter changes validated against safe ranges (e.g., max_daily_loss cannot be set to 0)
- Path parameters sanitized (no directory traversal in bot_id)

### CORS Policy (Dashboard)

```python
# Dashboard API CORS configuration
origins = [
    "http://localhost:5173",   # Vite dev server
    "http://localhost:8080",   # Production (same-origin)
]
# Production: same-origin only (SPA served by FastAPI)
```

---

## Infrastructure Security

### Network Segmentation

```
Internet
  │
  ├── SSH (port 22 or custom) ──→ VPS Host
  ├── HTTPS (port 443) ──→ Caddy/nginx ──→ Dashboard (8080)
  └── HTTPS (port 3000) ──→ Grafana (optional)

Docker Internal Network (polybot_net):
  ├── dashboard:8080    (exposed via reverse proxy)
  ├── postgres:5432     (internal only)
  ├── redis:6379        (internal only)
  ├── market-data       (internal only)
  ├── orchestrator      (internal only)
  ├── execution         (internal only)
  ├── risk              (internal only)
  ├── wallet            (internal only)
  ├── prometheus:9090   (internal only)
  └── grafana:3000      (optionally exposed)
```

### Secrets Management

| Secret | Storage | Rotation |
|--------|---------|----------|
| EOA private keys | .env encrypted with sops/age | Never (unless compromised) |
| L2 API credentials | Memory only (derived at startup) | Automatic on restart |
| Database password | .env / Docker secret | Quarterly |
| Dashboard API key | .env / Docker secret | Monthly (recommended) |
| Telegram bot token | .env | On compromise |

### VPS Hardening Checklist

```bash
# scripts/setup-vps.sh implements these
[ ] SSH: key-only authentication (PasswordAuthentication no)
[ ] SSH: non-standard port (e.g., 2222)
[ ] SSH: root login disabled (PermitRootLogin no)
[ ] Firewall: UFW enabled, allow only SSH + HTTPS + Grafana
[ ] Fail2Ban: installed and configured for SSH
[ ] Docker: user namespace remapping enabled
[ ] Docker: no privileged containers
[ ] Swap: encrypted or disabled (prevents key leakage to disk)
[ ] Updates: unattended-upgrades for security patches
[ ] Time sync: NTP configured (important for HMAC timestamps)
```

---

## Compliance Requirements

### Assessment

| Framework | Applicable? | Rationale |
|-----------|------------|-----------|
| **GDPR** | No | No personal user data collected; single-operator self-hosted platform |
| **SOC 2** | No | Not a SaaS product; no customer data |
| **HIPAA** | No | No health data |
| **PCI DSS** | No | No payment card processing; USDC is not card payment |
| **Polymarket ToS** | **Yes** | Must comply with geoblocking, no market manipulation, no oracle abuse |

### Polymarket-Specific Compliance

- **Geoblock check**: Platform checks Polymarket's geoblock API at startup; refuses to trade if VPS IP is in a restricted jurisdiction
- **Rate limit compliance**: Client-side rate limiting prevents overloading Polymarket's API (which can lead to blacklisting)
- **Balance/allowance integrity**: Orders never submitted without verified balance (Polymarket monitors for abuse and can blacklist)
- **No manipulation**: Strategies limited to legitimate trading (arbitrage, market making, value trading) — no wash trading, no oracle manipulation

---

## Security Development Lifecycle

### Static Analysis (SAST)

| Tool | Coverage | Integration |
|------|----------|-------------|
| **ruff** | Python linting, security-adjacent rules (no `eval`, no `exec`) | Pre-commit, CI |
| **mypy** (strict) | Type safety — prevents many categories of bugs | Pre-commit, CI |
| **pip-audit** | Python dependency vulnerability scanning | CI (weekly) |
| **npm audit** | Frontend dependency vulnerability scanning | CI (weekly) |
| **trivy** | Docker image vulnerability scanning | CI (on build) |

### Code Review Security Checklist

For every PR that touches these areas, verify:

- [ ] **Private keys**: No hardcoded keys; no keys in logs; no keys in error messages
- [ ] **Order submission**: Pre-trade risk checks not bypassed; fee rate included
- [ ] **Config changes**: Validated by Pydantic schema; logged in audit trail
- [ ] **New dependencies**: Checked for known vulnerabilities; justified
- [ ] **Error handling**: Sensitive data not leaked in exception messages
- [ ] **Dashboard endpoints**: Authentication required; input validated

---

## Incident Response

### Severity Levels

| Level | Definition | Response Time | Example |
|-------|-----------|---------------|---------|
| **SEV-1** | Active fund loss or key compromise | Immediate | Private key exposed; unauthorized trades executing |
| **SEV-2** | System down, no trading possible | 30 minutes | All services crashed; Polymarket API unreachable for >10 min |
| **SEV-3** | Degraded performance, partial functionality | 4 hours | One bot crashed; WebSocket reconnecting |
| **SEV-4** | Minor issue, no immediate impact | 24 hours | Dashboard slow; log rotation failed |

### Emergency Response Procedures

**SEV-1: Key Compromise**
1. Trigger Emergency Stop (dashboard button OR Redis: `SET emergency_stop 1`)
2. Transfer all funds from compromised wallet to Sweep wallet immediately
3. Generate new EOA private key
4. Rotate Polymarket API credentials: `create_or_derive_api_creds()` with new wallet
5. Update .env with new key
6. Investigate breach vector (SSH logs, Docker logs, file access times)
7. Post-mortem within 24 hours

**SEV-1: Oracle Manipulation Detected**
1. Check resolution proposals for affected markets
2. Close positions in affected markets if still possible
3. Do not trade on the affected event
4. Document the incident for future avoidance heuristics
5. Adjust per-market exposure caps if pattern indicates systemic risk

**SEV-2: Complete System Failure**
1. SSH into VPS; check Docker: `docker compose ps`
2. Review logs: `docker compose logs --tail=200`
3. If services crashed: `docker compose restart`
4. If database corrupted: restore from latest backup (`scripts/restore.sh`)
5. Verify all positions via Polymarket Data API (`/positions`)
6. Reconcile internal state with on-chain state
7. Resume trading only after full reconciliation

---

## Security Checklist (Pre-Launch)

- [ ] All private keys encrypted at rest (sops/age)
- [ ] .env file has 600 permissions, owned by deploy user
- [ ] .env.example committed; .env in .gitignore
- [ ] SSH key-only auth; root login disabled
- [ ] UFW firewall active; only SSH + HTTPS + Grafana open
- [ ] Fail2Ban configured for SSH
- [ ] Docker containers run as non-root user
- [ ] No internal ports (Redis, PostgreSQL) exposed to host
- [ ] Dashboard API key authentication implemented and tested
- [ ] Log redaction processor active (no keys in logs)
- [ ] pip-audit and npm audit show no critical vulnerabilities
- [ ] Emergency stop tested end-to-end (<5 seconds)
- [ ] Backup script tested; restore verified on fresh environment
- [ ] Geoblock compliance check implemented and tested
- [ ] Polymarket balance/allowance checks prevent over-trading
- [ ] NTP time sync configured (required for HMAC timestamp validity)

---

## Cross-References

| Topic | Document |
|-------|----------|
| Wallet architecture and key management | [04-technical-specification.md](./04-technical-specification.md) — Service 5 |
| Infrastructure deployment details | [09-infrastructure-spec.md](./09-infrastructure-spec.md) |
| Risk management design | [04-technical-specification.md](./04-technical-specification.md) — Service 4 |
| Oracle manipulation risk | [01-product-research.md](./01-product-research.md) — Risk Assessment |
