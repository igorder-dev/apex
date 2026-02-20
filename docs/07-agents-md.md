# Agent & Skill Configuration: PolyBot Platform

## AGENTS.md

> Copy to project root as `AGENTS.md` for Claude Code / opencode / Cursor / Windsurf multi-agent support.

```markdown
# AGENTS.md — PolyBot Platform

## Project Documentation

This project has a complete specification in the `docs/` folder (13 documents). **Every agent MUST consult the relevant documentation before implementing.**

### Fast Documentation Lookup

Instead of reading entire docs, use this hierarchy:

1. **`docs/llms.txt`** — Section-level index (~3K tokens). Read this FIRST to find the exact section and line range.
2. **`/docs` skill** — Run `/docs "your query"` to auto-match and load the right sections.
3. **`mcp-local-rag`** — Use `query_documents` tool for semantic search across all docs.
4. **Direct file read with offset/limit** — Read only the specific section you need (line ranges are in llms.txt).

### Documentation Quick Reference

| Area | Read This First | Then This |
|------|----------------|-----------|
| **Any task** | `docs/04-technical-specification.md` (architecture, data model, service designs) | `docs/03-prd.md` (user stories, acceptance criteria) |
| **API / endpoints** | `docs/10-api-specification.md` (REST + SSE + Polymarket integration) | `docs/04-technical-specification.md` → Service 6 |
| **Risk / execution** | `docs/04-technical-specification.md` → Service 3 + 4 | `docs/08-security-spec.md` (threat model) |
| **Frontend / dashboard** | `docs/10-api-specification.md` (API contracts) | `docs/04-technical-specification.md` → Service 6 |
| **Infrastructure** | `docs/09-infrastructure-spec.md` (Docker, VPS, monitoring) | `docs/08-security-spec.md` (hardening) |
| **Testing** | `docs/11-testing-strategy.md` (test suites, coverage targets) | `docs/05-development-guidelines.md` (CI/CD) |
| **New bot strategy** | `docs/04-technical-specification.md` → Binary Arb section (reference impl) | `docs/02-product-roadmap.md` (phasing) |
| **Wallet management** | `docs/04-technical-specification.md` → Service 5 | `docs/02-product-roadmap.md` → Wallet Architecture |

### Rules for Using Documentation

1. **Read before implementing**: Always read the relevant doc section before writing code. The docs contain exact specifications — don't guess.
2. **Docs are the source of truth**: If code contradicts docs, ask the user which is correct. Docs take precedence for new implementations.
3. **Cross-reference**: Each doc has a "Cross-References" section at the bottom pointing to related documents. Follow these links when your task spans multiple areas.
4. **Don't modify docs**: Agents should not edit `docs/` files unless the user explicitly requests documentation updates.

## Agent Roles

### core-architect
Focus: `src/core/`, `src/shared/`
Owns: BaseBot interface, Pydantic models, config loader, exceptions, enums, Redis client, DB engine.
Docs: `docs/04-technical-specification.md` → Data Model, Technology Stack, ADR-006
Rules:
- Changes to `bot_interface.py` require updating ALL bot implementations in `src/bots/`
- Pydantic models are the inter-service contract — backward compatibility required
- All new models need Alembic migrations if they touch DB tables

### service-dev
Focus: `src/services/`
Owns: All 6 services (market_data, orchestrator, execution, risk, wallet, dashboard).
Docs: `docs/04-technical-specification.md` → Service Architecture (Services 1-6), `docs/10-api-specification.md` → Inter-Service Communication
Rules:
- Every service has its own Dockerfile — changes to shared code must not break other services
- Inter-service communication ONLY through Redis (Streams, Pub/Sub, Cache) — no direct imports between services
- Each service entrypoint follows: `async def main()` with SIGTERM/SIGINT handlers

### bot-dev
Focus: `src/bots/`
Owns: Trading strategy modules.
Docs: `docs/04-technical-specification.md` → Binary Arb Strategy (reference impl), BaseBot interface
Rules:
- Must implement ALL BaseBot methods — orchestrator validates at load time
- Strategy logic lives in `strategy.py`, not in `bot.py` (bot.py is lifecycle glue)
- Never import from `src/services/` directly — use the injected BotContext
- Every bot must have unit tests for its strategy with realistic order book fixtures

### frontend-dev
Focus: `frontend/`
Owns: React dashboard SPA.
Docs: `docs/10-api-specification.md` → Dashboard API (all endpoints + SSE events), `docs/04-technical-specification.md` → Service 6
Rules:
- TypeScript types in `lib/types.ts` must match Pydantic models in `src/core/models.py`
- Use shadcn/ui components — don't write custom UI primitives
- SSE hook (`useSSE.ts`) handles reconnection automatically — don't create new EventSource instances
- All API calls go through `lib/api.ts` — no inline fetch() calls

### infra-ops
Focus: `docker-compose*.yml`, `config/`, `scripts/`, `Makefile`, `.github/`
Owns: Docker configuration, CI/CD, VPS setup, monitoring.
Docs: `docs/09-infrastructure-spec.md` (full Docker Compose, Caddy, Prometheus, backup), `docs/08-security-spec.md` (hardening)
Rules:
- Every Docker service needs a health check
- Resource limits are mandatory in docker-compose.yml (CPU + memory)
- Never expose internal ports (Redis, PostgreSQL) to the host network
- Prometheus scrape targets auto-discovered via Docker labels
```

---

## Claude Code Custom Slash Commands

### `/new-bot` — Scaffold a New Bot Module

```markdown
# /new-bot — Create a new trading strategy bot

When the user runs `/new-bot {strategy_name}`, scaffold the following:

1. Read `docs/04-technical-specification.md` → BaseBot interface section for the contract
2. Create directory: `src/bots/{strategy_name}/`
3. Create `__init__.py` (empty)
4. Create `bot.py` with class `{StrategyName}Bot(BaseBot)` implementing all 10 required methods
5. Create `strategy.py` with a placeholder strategy class
6. Create `models.py` with strategy-specific Pydantic models
7. Create `config/bots/{strategy_name}-default.yaml` with the standard config schema
8. Add a unit test file: `tests/unit/test_{strategy_name}_strategy.py`

Use the existing `src/bots/binary_arbitrage/` as the reference implementation for structure.
Ensure the bot class imports from `src.core.bot_interface` and all hooks are properly typed.
```

### `/check-risk` — Audit Risk Configuration

```markdown
# /check-risk — Validate risk parameters

When the user runs `/check-risk`:

1. Read `docs/04-technical-specification.md` → Service 4: Risk Manager for the 10-step pipeline spec
2. Read `config/risk.yaml` and validate all values are within safe ranges
3. Read all `config/bots/*.yaml` and verify per-bot risk params don't exceed global limits
4. Check that per-bot virtual balances don't exceed their wallet's total capacity
5. Verify circuit breaker thresholds are configured for every enabled bot
6. Report any gaps, inconsistencies, or dangerous configurations
```

### `/test-strategy` — Run Strategy Unit Tests

```markdown
# /test-strategy {bot_name}

When the user runs `/test-strategy {bot_name}`:

1. Read `docs/11-testing-strategy.md` → Critical Unit Test Suites for coverage expectations
2. Run: `pytest tests/unit/test_{bot_name}_strategy.py -v --tb=long`
3. If tests pass, also run: `pytest tests/integration/ -k {bot_name} -v`
4. Report coverage for the strategy module: `pytest --cov=src/bots/{bot_name} --cov-report=term-missing`
```

### `/deploy` — Build and Deploy to VPS

```markdown
# /deploy — Production deployment checklist

When the user runs `/deploy`:

1. Read `docs/09-infrastructure-spec.md` → Deployment Workflow for the full procedure
2. Run all tests: `make test`
3. Build frontend: `cd frontend && npm run build`
4. Build Docker images: `docker compose build`
5. Show diff of config changes since last deploy
6. Prompt user to confirm deployment
7. If confirmed: `docker compose up -d && docker compose logs -f --tail=50`
```

---

## Sub-Agent Definitions

For complex tasks, use these sub-agent configurations:

### Market Data Agent

```yaml
# docs/.claude/agents/market-data-agent.md → .claude/agents/market-data-agent.md
name: market-data-agent
description: Implements and debugs the Market Data Service
focus_dirs:
  - src/services/market_data/
  - src/core/models.py
  - src/shared/redis_client.py
context_files:
  - docs/04-technical-specification.md  # Service 1 section
  - docs/10-api-specification.md        # Polymarket API integration + Redis Streams schema
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
instructions: |
  You are responsible for the Market Data Service.

  BEFORE WRITING CODE: Read docs/04-technical-specification.md → Service 1: Market Data Service
  for the full design spec. Read docs/10-api-specification.md → Polymarket API Integration for
  API contracts and WebSocket payload formats.

  Key constraints:
  - WebSocket max 500 instruments per connection
  - No unsubscribe support — new connections for new subscription sets
  - Publish to Redis Stream market_data:{token_id}
  - Cache latest orderbook in Redis Hash orderbook:{token_id}
  - Gamma API polling every 60s for market discovery
  - Exponential backoff reconnection (1s→30s, max 10 retries)
```

### Execution Agent

```yaml
# docs/.claude/agents/execution-agent.md → .claude/agents/execution-agent.md
name: execution-agent
description: Implements and debugs the Execution Engine
focus_dirs:
  - src/services/execution/
  - src/services/wallet/
  - src/core/models.py
context_files:
  - docs/04-technical-specification.md  # Service 3 + Service 5 sections
  - docs/10-api-specification.md        # CLOB REST API authentication + order submission
  - docs/08-security-spec.md            # Key management, HMAC signing
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
instructions: |
  You are responsible for the Execution Engine and Wallet Manager integration.

  BEFORE WRITING CODE: Read docs/04-technical-specification.md → Service 3: Execution Engine
  and Service 5: Wallet Manager for the full design. Read docs/10-api-specification.md →
  Polymarket CLOB REST API for authentication headers and order submission.

  Key constraints:
  - Wrap py-clob-client (v0.34.5) — don't use it directly elsewhere
  - Rate limit: token bucket 3,500/10s PER WALLET
  - Batch orders: up to 15 per post_orders() call
  - FOK decimal precision: sell ≤2 decimals, taker ≤4, size×price ≤2
  - Fee rates are dynamic — always check fee_cache before submission
  - Route orders to correct wallet API key via bot→tier mapping
  - Per-wallet mutex to prevent concurrent order race conditions
```

### Risk Agent

```yaml
# docs/.claude/agents/risk-agent.md → .claude/agents/risk-agent.md
name: risk-agent
description: Implements and debugs the Risk Manager
focus_dirs:
  - src/services/risk/
  - src/core/models.py
  - config/risk.yaml
context_files:
  - docs/04-technical-specification.md  # Service 4 section
  - docs/11-testing-strategy.md         # Risk-specific test suites and coverage requirements
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
instructions: |
  You are responsible for the Risk Manager. This is the most critical service — bugs here
  mean unbounded losses.

  BEFORE WRITING CODE: Read docs/04-technical-specification.md → Service 4: Risk Manager for
  the full design. Read docs/11-testing-strategy.md → Critical Unit Test Suites for the
  pre_trade_checks and circuit_breaker test specifications.

  Key constraints:
  - 10-step pre-trade risk pipeline: emergency flag → circuit breaker → daily P&L → market position → portfolio exposure → rate limit → wallet balance → price sanity → drawdown protection → per-trade max loss
  - Circuit breaker FSM: CLOSED → OPEN → HALF_OPEN → CLOSED (or PERMANENTLY_STOPPED)
  - Emergency stop must cancel ALL orders across ALL wallets in <5 seconds
  - Risk hierarchy: Global > Strategy > Bot > Market
  - Every rejection must be logged with the specific check that failed
  - NEVER skip a risk check. NEVER allow an order through without all 10 checks passing.
```

### Dashboard Agent

```yaml
# docs/.claude/agents/dashboard-agent.md → .claude/agents/dashboard-agent.md
name: dashboard-agent
description: Implements the Dashboard API and React frontend
focus_dirs:
  - src/services/dashboard/
  - frontend/src/
context_files:
  - docs/10-api-specification.md        # All REST + SSE endpoints with request/response schemas
  - docs/04-technical-specification.md  # Service 6 section
  - docs/11-testing-strategy.md         # Frontend test requirements
tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
instructions: |
  You are responsible for the Dashboard (FastAPI backend + React frontend).

  BEFORE WRITING CODE: Read docs/10-api-specification.md for the complete API contract —
  every endpoint, request/response schema, SSE event format, and error codes are specified
  there. Read docs/04-technical-specification.md → Service 6 for SSE publish intervals and
  route groups.

  Key constraints:
  - All REST endpoints require Bearer token auth (DASHBOARD_API_KEY)
  - SSE events published via Redis Pub/Sub dashboard_events channel
  - React SPA uses shadcn/ui — no custom UI primitives
  - TypeScript types in lib/types.ts must match Pydantic models in src/core/models.py
  - All API calls go through lib/api.ts — no inline fetch()
  - SSE hook (useSSE.ts) handles reconnection — don't create new EventSource instances
```

### `/debug-bot` — Comprehensive Bot Diagnostic

```markdown
# /debug-bot {bot_id_or_name}

When the user runs `/debug-bot {bot_id}`:

1. Check bot status in orchestrator (Redis: GET bot_status:{bot_id})
2. Check container health (Docker MCP)
3. Query recent signals from PostgreSQL (last 20)
4. Query recent fills from PostgreSQL (last 20)
5. Query risk rejections from PostgreSQL (last 10)
6. Check circuit breaker state (Redis: GET circuit_breaker:{bot_id})
7. Check market data stream health (Redis: XINFO STREAM)
8. Check consumer group lag (Redis: XINFO GROUPS)
9. Check wallet balance (Redis: HGETALL wallet_balance:{wallet_id})
10. Query Prometheus metrics (if Grafana MCP available)
11. Read recent container logs (Docker MCP)

Produce a structured diagnostic report with severity levels and recommended actions.
Requires: postgres, redis, docker MCP servers.
```

### `/e2e-test` — Dashboard E2E Tests

```markdown
# /e2e-test [suite_name]

When the user runs `/e2e-test [dashboard|emergency-stop|config-change|all]`:

1. Read `docs/11-testing-strategy.md` → E2E test specifications
2. Verify dashboard is accessible via Playwright MCP
3. Run the specified test suite using Playwright MCP:
   - dashboard: verify bot cards, charts, SSE updates
   - emergency-stop: trigger stop, verify all bots STOPPED
   - config-change: modify config via UI, verify propagation
4. Take screenshots at checkpoints
5. Report pass/fail with screenshots

Requires: Playwright MCP (Claude Code plugin).
```

### `/wallet-audit` — Financial Reconciliation

```markdown
# /wallet-audit

When the user runs `/wallet-audit`:

1. Read `docs/04-technical-specification.md` → Service 5: Wallet Manager
2. Query PostgreSQL: virtual balance sums vs wallet totals
3. Query PostgreSQL: pending orders vs available balances
4. Query Redis: cached balances vs database values
5. Verify tier allocations match config (Vault ~70%, Alpha ~25%, Sweep ~5%)
6. Check for orphaned ledger entries
7. Report discrepancies with severity: OK / WARN / CRITICAL

Requires: postgres, redis MCP servers.
```

### `/market-scan` — Polymarket Opportunity Scanner

```markdown
# /market-scan [--min-profit 0.005] [--min-liquidity 1000]

When the user runs `/market-scan`:

1. Read `docs/04-technical-specification.md` → Binary Arbitrage Strategy
2. Fetch active binary markets from Gamma API
3. Calculate spread and net profit after fees for each pair
4. Rank by expected profit, filter by thresholds
5. Present opportunities table with liquidity and resolution timing

Read-only — does not place orders.
```

### `/perf-profile` — Async Service Performance Profiler

```markdown
# /perf-profile {service_name}

When the user runs `/perf-profile {service_name}`:

1. Read `docs/05-development-guidelines.md` → Performance Budget
2. Query Prometheus metrics (if Grafana MCP available): latency percentiles, throughput, error rates
3. Check container resource usage (Docker MCP)
4. Check slow queries (PostgreSQL MCP)
5. Compare against performance budget targets
6. Report bottlenecks and optimization suggestions

Requires: docker MCP. Enhanced with grafana, postgres MCP servers.
```

### `/backtest` — Historical Strategy Backtesting

```markdown
# /backtest {bot_name} [--days 7] [--market "slug"]

When the user runs `/backtest {bot_name}`:

1. Read the bot's strategy.py algorithm
2. Query TimescaleDB for historical order book snapshots
3. Simulate signal generation against historical data
4. Calculate P&L with fees, slippage, and rate limits
5. Compare with actual results if available
6. Produce backtest report with P&L, win rate, max drawdown

Requires: postgres MCP. Enhanced with sequential-thinking MCP.
Available after historical data accumulates (Week 3+).
```

---

## MCP Server Configuration

### MCP Servers

All servers are configured in `docs/mcp-config.json` (copy to `.mcp.json` during setup). Full specification: `docs/14-mcp-skills-specification.md`.

```bash
# P0 — Required (install during initial setup)
# mcp-local-rag — Semantic search over project documentation (fully local, no API keys)
claude mcp add local-rag --scope project --env BASE_DIR=./docs -- npx -y mcp-local-rag

# Context7 — External library documentation lookup
claude mcp add context7 --scope project -- npx -y @upstash/context7-mcp

# PostgreSQL — Database inspection, query testing, EXPLAIN analysis
claude mcp add postgres --scope project -- npx -y @crystaldba/postgres-mcp

# Redis — Stream, cache, pub/sub inspection
claude mcp add redis --scope project -- npx -y @redis/mcp-redis

# Docker — Container management, logs, health checks
claude mcp add docker --scope project -- npx -y @quantgeekdev/docker-mcp

# P1 — Add after MVP is running (Week 7+)
# Grafana — Prometheus metrics via PromQL
# claude mcp add grafana --scope project -- npx -y @grafana/mcp-grafana

# Sequential Thinking — Complex strategy/risk analysis
# claude mcp add sequential-thinking --scope project -- npx -y @modelcontextprotocol/server-sequential-thinking

# Full setup instructions: docs/13-claude-code-setup.md
```

### When to Use Which Tool

| Need | Use |
|------|-----|
| "Which doc/section covers X?" | Read `docs/llms.txt` (section-level index) |
| "Show me the risk manager design" | `/docs risk manager` skill |
| "How does the order state machine handle timeouts?" | `mcp-local-rag` → `query_documents` (semantic search) |
| "What's the py-clob-client API for batch orders?" | Context7 → `resolve-library-id` then `get-library-docs` |
| "Implement Service 3 from scratch" | Direct read of `docs/04-technical-specification.md` Service 3 section |
| "What's in the market data stream?" | Redis MCP → `XINFO STREAM market_data:{token_id}` |
| "Show recent fill data" | PostgreSQL MCP → `SELECT * FROM fill ORDER BY created_at DESC LIMIT 20` |
| "Is the orchestrator container healthy?" | Docker MCP → container status/health |

**Rule of thumb**:
- Use `docs/llms.txt` + `/docs` skill for *PolyBot's own design*
- Use `mcp-local-rag` for *fuzzy/semantic queries* across project docs
- Use Context7 for *external library APIs* (py-clob-client, FastAPI, SQLAlchemy, Redis, etc.)
- Use PostgreSQL/Redis/Docker MCPs for *live system inspection and debugging*

---

## Development Workflow with Agents

### Implementing a New Service Feature

```
1. Read docs/04-technical-specification.md → relevant service section
2. Read docs/03-prd.md → find the user story with acceptance criteria
3. Read docs/10-api-specification.md if the feature exposes or consumes an API
4. Implement in the service directory (src/services/{name}/)
5. Write unit tests (see docs/11-testing-strategy.md for critical test suites)
6. Run: make test-unit
7. Write integration test if it crosses service boundaries
8. Run: make test-int
9. Update Prometheus metrics if new observables added
10. Update dashboard API if new data exposed
11. ⏸ DEMO TO USER: summarize changes, list what to validate, wait for approval
12. On approval: commit with conventional message
13. Update STATE.md: mark feature complete (add commit hash), advance to next feature
14. Commit STATE.md: `chore(state): update project state after US-XXX`
```

### Implementing a New Bot Strategy

```
1. Run /new-bot {strategy_name} to scaffold
2. Read docs/04-technical-specification.md → Binary Arb Strategy section as reference
3. Read docs/04-technical-specification.md → BaseBot ABC for the interface contract
4. Implement strategy.py with the core algorithm
5. Implement bot.py lifecycle hooks (delegate to strategy.py)
6. Create config/bots/{name}-default.yaml with paper_trading: true, graduated: false
7. Write unit tests with realistic order book fixtures (see docs/11-testing-strategy.md)
8. ⏸ DEMO TO USER: summarize bot design, show test results, wait for approval
9. Run in paper trading mode ≥48h (see docs/11-testing-strategy.md → Paper Trading)
10. ⏸ SHOW USER paper trading results. User decides: approve graduation or continue paper
11. If approved: user sets graduated: true in bot config. Never do this without user instruction.
12. After each commit in this workflow, update STATE.md with current step and commit hash
13. Commit STATE.md: `chore(state): update project state`
```

### Modifying Infrastructure

```
1. Read docs/09-infrastructure-spec.md → relevant section (Docker, VPS, monitoring, backup)
2. Read docs/08-security-spec.md → if change affects network, secrets, or access
3. Implement changes
4. Verify health checks pass: make health
5. Test deployment: make deploy (or docker compose up -d)
6. ⏸ DEMO TO USER: summarize infra changes, show health check results, wait for approval
7. On approval: commit with conventional message
8. Update STATE.md and commit: `chore(state): update project state after infra change`
```

**Critical rule for all workflows**: Steps marked with ⏸ are mandatory human-in-the-loop checkpoints. Agents must present their work and wait for explicit user approval before proceeding. See `docs/12-project-governance.md` → Agent Feature Delivery Protocol for the demo template.

### STATE.md Maintenance Rule

**Every commit must be followed by a STATE.md update.** The update can be:
- **Combined**: Include STATE.md in the same commit as the code (preferred for small changes)
- **Separate**: Follow-up `chore(state): update project state after [description]` commit

Format: `chore(state): update project state after [brief description]`

If STATE.md is stale (last commit doesn't match STATE.md's "Last commit" field), fix it immediately before starting new work. See `docs/15-state-template.md` for the STATE.md format.

---

## Compatibility Notes

### Claude Code
- `CLAUDE.md` in project root is auto-loaded as system context
- `AGENTS.md` in project root defines agent roles for multi-agent workflows
- Sub-agents in `docs/.claude/agents/*.md` (copy to `.claude/agents/` during setup) for focused tasks (market-data, execution, risk, dashboard)
- 10 slash commands: `/new-bot`, `/check-risk`, `/test-strategy`, `/deploy`, `/debug-bot`, `/e2e-test`, `/wallet-audit`, `/market-scan`, `/perf-profile`, `/backtest`
- `/docs` skill available for targeted documentation lookup
- 5 MCP servers (P0): `local-rag`, `context7`, `postgres`, `redis`, `docker`
- 2 MCP servers (P1): `grafana`, `sequential-thinking`
- Playwright MCP connected as Claude Code plugin for E2E testing
- **Full setup**: See `docs/13-claude-code-setup.md`
- **MCP & skills spec**: See `docs/14-mcp-skills-specification.md`

### opencode
- `CLAUDE.md` is read as project instructions (same as Claude Code)
- Agent roles from `AGENTS.md` provide context for task delegation
- `docs/llms.txt` provides a section-level index for efficient doc navigation
- The `docs/` documentation reference table in CLAUDE.md guides the agent to the right specification files

### Cursor / Windsurf
- `AGENTS.md` defines focus areas and rules per agent role
- `.cursor/rules` or `.windsurfrules` can be derived from the agent roles above
- `docs/llms.txt` provides a section-level index for efficient doc navigation
- The documentation reference works the same — agents should read `docs/` before implementing
