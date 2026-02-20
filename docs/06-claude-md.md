# Generated CLAUDE.md for PolyBot Platform

> Copy this file to the project root as `CLAUDE.md`

---

# CLAUDE.md — PolyBot Platform

Automated trading platform for Polymarket prediction markets. Python 3.11+ async services with Docker Compose deployment on VPS.

## Feature Delivery Protocol — MANDATORY

**Every feature must follow this cycle. Do NOT skip user validation.**

1. Pick feature → Read docs (via `docs/llms.txt`) → Implement + tests → Run `make test-unit`
2. **Demo to user**: Summarize what was built, what tests pass, and list specific things for user to validate
3. **Wait for user approval** before committing or moving to next feature
4. If user finds issues → fix → demo again

**Full protocol details**: `docs/12-project-governance.md` → Agent Feature Delivery Protocol

## Session Start Protocol — MANDATORY

**Every new session begins with these steps before any work.**

1. **Read `STATE.md`** (project root) — Current phase, in-progress feature, completed work, blockers
2. **Run `git log --oneline -5`** — Cross-reference with STATE.md to confirm what was committed

After reading both, announce your orientation:

```
SESSION RESUMED:
  Current: [feature from STATE.md]
  Last commit: [hash — description]
  Next action: [what you'll do first]
```

If `STATE.md` does not exist, create it from `docs/15-state-template.md` and populate from `docs/02-product-roadmap.md` (current phase) and `docs/03-prd.md` (next user stories).

**On every commit**: Update STATE.md — mark completed work, advance in-progress, refresh next steps. See `docs/12-project-governance.md` → Agent Feature Delivery Protocol step 9.

## Documentation — READ FIRST

The `docs/` folder contains the complete project specification. **Before implementing any feature, consult the relevant document.**

| Doc | What It Covers | When to Read |
|-----|---------------|--------------|
| `docs/04-technical-specification.md` | Architecture, ADRs, data model (ER diagram), all 6 service designs, Redis topology, binary arb algorithm | **Every task** — this is the source of truth for how the system works |
| `docs/03-prd.md` | 63 user stories with acceptance criteria (Given/When/Then) | Before implementing any user-facing feature |
| `docs/10-api-specification.md` | Dashboard REST + SSE endpoints, Polymarket API integration, inter-service Redis schemas, Pydantic models | When working on APIs, dashboard, or service communication |
| `docs/05-development-guidelines.md` | Repo structure, dev setup, Git workflow, code standards, CI/CD, performance budget | First-time setup, adding dependencies, CI questions |
| `docs/08-security-spec.md` | Threat model, auth design, log redaction patterns, secrets handling | When touching auth, keys, logging, or network config |
| `docs/09-infrastructure-spec.md` | Docker Compose configs, VPS setup, Caddy, Prometheus/Grafana, backup/recovery | When changing Docker, deployment, or monitoring |
| `docs/11-testing-strategy.md` | Test pyramid, critical test suites with examples, coverage targets, paper trading checklist | When writing or modifying tests |
| `docs/02-product-roadmap.md` | 6-phase roadmap, wallet architecture, decision log | Understanding what's in scope for each phase |
| `docs/01-product-research.md` | Market research, Polymarket ecosystem, risk assessment, competitor analysis | Understanding the product domain |
| `docs/12-project-governance.md` | Contribution workflow, PR template, release process, post-mortem template | Process questions, team scaling |

**Rule**: If you're unsure how something should work, the answer is in `docs/`. Read the relevant document before guessing. The tech spec (doc 04) and API spec (doc 10) are the most frequently needed.

### Fast Documentation Lookup

Instead of reading entire docs, use this hierarchy:

1. **`docs/llms.txt`** — Section-level index (~3K tokens). Read this first to find the exact section and line range you need.
2. **`/docs` skill** — Run `/docs "your query"` to auto-match and load the right sections.
3. **`mcp-local-rag`** — Use `query_documents` tool for semantic search when you're unsure which section to check.
4. **Direct file read** — Only read full documents when implementing a complete service from scratch.

**Setup**: See `docs/13-claude-code-setup.md` for full configuration of skills and MCP servers.

## Build / Run / Test

```bash
# Dev mode (all services with hot reload)
make dev

# Production
docker compose up -d

# Tests
make test               # All tests
make test-unit          # Unit only (fast, no Docker deps)
make test-int           # Integration (requires postgres + redis)
pytest tests/unit/test_circuit_breaker.py -v  # Single file

# Lint + type check
ruff check src/
mypy src/

# Database
alembic upgrade head    # Run migrations
alembic revision --autogenerate -m "description"  # New migration

# Frontend
cd frontend && npm run dev    # Dev server
cd frontend && npm run build  # Production build
```

## Architecture — 6 Services + Bot Modules

```
Orchestrator → loads bot modules → bots emit Signals
  → Execution Engine (rate-limited, wallet-routed) → Polymarket CLOB API
  → Risk Manager (pre-trade checks, circuit breakers) gates every order
  → Wallet Manager (multi-EOA, software ledger) routes to correct wallet
  → Market Data Service (WebSocket + Gamma) → Redis Streams → bots
  → Dashboard API (FastAPI + React SPA) → SSE to browser
```

All inter-service communication goes through **Redis 7** (Streams for durable messages, Pub/Sub for commands, Hash for caches). Database is **PostgreSQL 16 + TimescaleDB**.

**Full architecture diagrams**: See `docs/04-technical-specification.md` — System Context Diagram + High-Level Architecture.

## Key Directories

- `docs/` — **Complete project specification (13 documents).** Read `docs/llms.txt` first for the section-level index.
- `src/core/` — BaseBot ABC, shared Pydantic models, config loader, enums. **DO NOT modify bot_interface.py without updating ALL bots.**
- `src/services/` — 6 microservices (market_data, orchestrator, execution, risk, wallet, dashboard). Each has its own Dockerfile.
- `src/bots/` — Strategy modules. Each bot is a directory with `bot.py` (implements BaseBot), `strategy.py` (algorithm), `models.py`.
- `src/shared/` — Redis client, DB engine, metrics, logging, Telegram alerts. Shared across all services.
- `config/bots/` — YAML configs per bot instance. Validated by Pydantic on load.
- `config/wallets.yaml` — Wallet tier definitions (Vault/Alpha/Sweep).
- `config/risk.yaml` — Global risk parameters.
- `frontend/src/` — React 18 + shadcn/ui + Recharts dashboard.

## Critical Patterns

### Bot Interface (every bot must implement)

```python
class MyBot(BaseBot):
    async def on_init(self, ctx: BotContext) -> None: ...
    async def on_start(self) -> None: ...
    async def on_market_data(self, token_id: str, data: OrderBookSnapshot) -> None: ...
    async def on_fill(self, fill: FillEvent) -> None: ...
    async def on_pause(self) -> None: ...
    async def on_resume(self) -> None: ...
    async def on_stop(self) -> None: ...
    async def on_emergency_stop(self) -> None: ...
    def get_metrics(self) -> dict[str, float]: ...
    def get_health(self) -> HealthStatus: ...
```

### Order Submission Flow

Signal → Pre-Trade Risk Checks (8 steps) → Wallet Router (bot→tier→API key) → Rate Limiter (3,500/10s per wallet) → Fee Rate Check → Order Build (py-clob-client) → Batch Submit (up to 15/call)

**Full flow diagram**: `docs/04-technical-specification.md` → Service 3: Execution Engine.

### Wallet Architecture

- 1 EOA = 1 proxy wallet (Polymarket CREATE2 constraint)
- Risk tiers: Vault (low-risk bots, ~70% capital), Alpha (high-risk, ~25%), Sweep (cold, ~5%)
- Software Ledger in PostgreSQL tracks per-bot P&L within shared wallets
- Per-wallet mutex prevents concurrent order race conditions

**Full wallet design**: `docs/04-technical-specification.md` → Service 5: Wallet Manager.

## Gotchas & Non-Obvious Conventions

1. **FOK decimal precision**: Sell maker amounts ≤ 2 decimals, taker amounts ≤ 4 decimals, size × price ≤ 2 decimals. Polymarket silently rejects orders violating this. Validation in `execution/clob_client_wrapper.py`.

2. **WebSocket 500-instrument limit**: Max 500 subscriptions per connection. `websocket_manager.py` spawns additional connections. **No unsubscribe support** — new subscription sets require new connections.

3. **Fee rates are dynamic**: Some markets (15-min crypto) have taker fees up to 3.15%. Always use `fee_cache.py`, never hardcode 0. Fee rate must be included in signed order data.

4. **Rate limits are per-wallet, not global**: Each wallet gets its own 3,500/10s token bucket. Batch orders (up to 15) consume 1 token.

5. **Redis Streams are the backbone**: All market data flows through `market_data:{token_id}` streams. Bots consume via consumer groups. If a bot misses messages (was paused), it catches up from the stream's backlog.

6. **Emergency stop flag**: Redis key `emergency_stop`. Every service checks this flag on every operation cycle. Set to "1" to halt everything. Manual clear required to resume.

7. **Pydantic v2 models are the contract**: Never pass raw dicts between services. Always use models from `src/core/models.py`. This is enforced by mypy strict mode.

8. **TimescaleDB hypertables**: `fill`, `signal`, `bot_metric`, `order_book_snapshot` tables are hypertables. Use `time_bucket()` for aggregation queries, not `date_trunc()`.

9. **Alembic + TimescaleDB**: When creating new hypertables in migrations, call `select create_hypertable(...)` AFTER the table creation. Alembic autogenerate won't detect hypertable conversion.

10. **py-clob-client API key derivation is idempotent**: `create_or_derive_api_creds()` returns the same credentials for the same EOA every time. Cache them at startup, don't call repeatedly.

11. **Debug mode generates 100x more log volume**: `debug_mode: true` in bot config logs every signal evaluation, every risk check (all 10), every order decision with reasoning. Enable only for specific bots during analysis, not globally. Logs go to `logs/{bot_id}_debug.jsonl` (500MB cap). See `docs/04-technical-specification.md` → Bot Debug Mode.

12. **New bots ALWAYS start in paper mode**: `paper_trading: true` and `graduated: false` are required defaults. The Orchestrator refuses to start a bot with `paper_trading: false` AND `graduated: false`. Only the operator can set `graduated: true` after the paper trading validation checklist passes. See `docs/11-testing-strategy.md` → Paper Trading.

13. **Pre-trade risk pipeline is 10 steps, not 8**: Two additional checks were added — drawdown protection (step 9) and per-trade max loss (step 10). Never skip any check. See `docs/04-technical-specification.md` → Service 4.

## Environment Variables (.env)

```
VAULT_PRIVATE_KEY=         # EOA private key for Vault wallet (hex, no 0x prefix)
ALPHA_PRIVATE_KEY=         # EOA private key for Alpha wallet (Phase 3)
SWEEP_PRIVATE_KEY=         # EOA private key for Sweep wallet (Phase 2)
DATABASE_URL=              # postgresql+asyncpg://polybot:password@postgres:5432/polybot
REDIS_URL=                 # redis://redis:6379/0
TELEGRAM_BOT_TOKEN=        # Telegram alert bot token
TELEGRAM_CHAT_ID=          # Telegram chat ID for alerts
POLYMARKET_CLOB_URL=       # https://clob.polymarket.com
POLYMARKET_WS_URL=         # wss://ws-subscriptions-clob.polymarket.com/ws/
GAMMA_API_URL=             # https://gamma-api.polymarket.com
POLYGON_CHAIN_ID=137       # Polygon mainnet
DASHBOARD_API_KEY=         # API key for dashboard authentication
```

## Conventions

- Commit format: `type(scope): description` — types: feat, fix, refactor, test, docs, ci, perf, chore
- Branch naming: `{type}/{ticket-id}-{short-description}`
- All async: no sync DB calls, no sync Redis calls, no sync HTTP calls
- structlog JSON logging with correlation_id across all services
- Prometheus metrics prefixed with `polybot_`
- **Full conventions**: `docs/05-development-guidelines.md`

## MCP Servers

```bash
# Local RAG — semantic search over project docs (no API keys, fully offline)
# Configured in .mcp.json, auto-starts with Claude Code
# Tools: ingest_file, query_documents, list_files, delete_file, status

# Context7 — external library documentation lookup
# Tools: resolve-library-id, get-library-docs
```

**When to use which**: `mcp-local-rag` for *PolyBot's own design*. Context7 for *external library APIs* (py-clob-client, FastAPI, SQLAlchemy, Redis, etc.).

**Full setup instructions**: `docs/13-claude-code-setup.md`
