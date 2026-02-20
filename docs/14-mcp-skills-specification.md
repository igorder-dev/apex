# MCP Servers & Skills Specification: PolyBot Platform

## Overview

| Field | Value |
|-------|-------|
| Product | PolyBot Platform |
| Author | Apex |
| Status | Draft |
| Last Updated | 2026-02-17 |
| Scope | Development & operations tooling for Claude Code |

This document specifies all MCP (Model Context Protocol) servers and Claude Code skills/commands for developing and operating the PolyBot platform. It extends the base tooling (local-rag + context7 + /docs skill) defined in `docs/07-agents-md.md` and `docs/13-claude-code-setup.md` with development, debugging, and operations capabilities.

---

## Design Principles

1. **Don't over-tool**: Each MCP server consumes context window (~800-2,000 tokens). Only add servers that provide frequent, concrete value.
2. **Phase-gated rollout**: P0 servers are needed from day one. P1 servers add value once services are running. Don't install everything upfront.
3. **Skills wrap MCPs**: Slash commands orchestrate multiple MCP tools into domain-specific workflows (e.g., `/debug-bot` uses PostgreSQL + Redis + Docker MCPs together).
4. **Existing tools first**: Claude Code has built-in Read/Write/Edit/Glob/Grep/Bash. MCP servers add value only where built-in tools are insufficient.

---

## MCP Server Inventory

### Existing Servers (from initial setup)

#### 1. local-rag — Project Documentation Search

| Field | Value |
|-------|-------|
| Package | `mcp-local-rag` |
| Install | `claude mcp add local-rag --scope project --env BASE_DIR=./docs -- npx -y mcp-local-rag` |
| Purpose | Semantic search over all `docs/` files |
| Tools | `ingest_file`, `ingest_data`, `query_documents`, `list_files`, `delete_file`, `status` |
| When to use | Fuzzy/semantic queries across project documentation |

#### 2. context7 — External Library Documentation

| Field | Value |
|-------|-------|
| Package | `@upstash/context7-mcp` |
| Install | `claude mcp add context7 --scope project -- npx -y @upstash/context7-mcp` |
| Purpose | Current docs for py-clob-client, FastAPI, SQLAlchemy, Redis, Pydantic, React, shadcn/ui, etc. |
| Tools | `resolve-library-id` (call first), `get-library-docs` |
| When to use | Looking up external library APIs, verifying framework capabilities, getting code examples |

---

### P0 Servers — Required for MVP Development

#### 3. postgres — Database Inspection & Query Testing

| Field | Value |
|-------|-------|
| Package | `@crystaldba/postgres-mcp` |
| Install | `claude mcp add postgres --scope project -- npx -y @crystaldba/postgres-mcp` |
| Env | `DATABASE_URL=postgresql://polybot:password@localhost:5432/polybot` |
| Mode | `restricted` (read-only). Use `unrestricted` only in dev when DDL is needed |
| Purpose | Schema inspection, query testing, EXPLAIN analysis, index validation |

**Why it's essential for PolyBot**:
- 15+ database entities with complex relationships (bots, wallets, fills, signals, orders, positions, ledger_entries)
- TimescaleDB hypertables require specific query patterns (`time_bucket()` not `date_trunc()`) that need validation
- Software ledger queries involve complex aggregations across multiple tables
- EXPLAIN plan analysis is critical: `fill` and `signal` hypertables grow fast; query performance must be validated early
- Schema inspection after Alembic migrations prevents Pydantic model ↔ DB schema drift

**Key tools**: `query` (run SQL), `schema` (inspect tables), `explain` (analyze query plans), `indexes` (list indexes)

**Safety rules**:
- NEVER connect to production in `unrestricted` mode
- Connection string must use dev-only credentials
- Destructive DDL (DROP, TRUNCATE) requires explicit user confirmation

---

#### 4. redis — Stream, Cache & Pub/Sub Inspection

| Field | Value |
|-------|-------|
| Package | `@redis/mcp-redis` |
| Install | `claude mcp add redis --scope project -- npx -y @redis/mcp-redis` |
| Env | `REDIS_URL=redis://localhost:6379/0` |
| Purpose | Inspect Redis Streams, Pub/Sub channels, cache state, consumer groups |

**Why it's essential for PolyBot**:
- Redis is the **entire inter-service communication backbone** — every service talks through Redis
- Streams: `market_data:{token_id}` (market data), `execution_commands` (order flow), `orchestrator_commands` (bot lifecycle)
- Pub/Sub: `dashboard_events` (SSE broadcast), `emergency_stop` (global halt)
- Cache: `orderbook:{token_id}`, `fee_cache:{market_id}`, `wallet_balance:{wallet_id}`
- Consumer groups per bot on market data streams — lag inspection required for debugging
- Circuit breaker state, rate limiter tokens, emergency stop flag — all in Redis

**Key use cases**:
- `XINFO STREAM market_data:{token_id}` — check if market data is flowing
- `XINFO GROUPS market_data:{token_id}` — check consumer group lag per bot
- `GET emergency_stop` — verify emergency stop state
- `HGETALL orderbook:{token_id}` — inspect cached orderbook
- `XRANGE` with COUNT — read recent stream entries for debugging

**Safety rules**:
- NEVER use `FLUSHDB`, `FLUSHALL`, or `DEL` on production Redis
- Stream reads on high-volume keys should use COUNT limits to prevent large payloads

---

#### 5. docker — Container Management & Logs

| Field | Value |
|-------|-------|
| Package | `@quantgeekdev/docker-mcp` |
| Install | `claude mcp add docker --scope project -- npx -y @quantgeekdev/docker-mcp` |
| Purpose | Container management, log inspection, health check monitoring |

**Why it's essential for PolyBot**:
- PolyBot runs **11+ Docker containers** (6 services + PostgreSQL + Redis + Caddy + Prometheus + Grafana)
- Every development session involves container operations: rebuild a service, tail logs, check health, restart
- Container health checks are defined for every service — need verification
- The `/deploy` command requires container management capabilities
- Log inspection across multiple containers is the primary debugging method

**Key use cases**:
- List running containers and their status/health
- Read container logs (with filtering by service name)
- Restart individual services after code changes
- Check resource usage (CPU, memory) per container
- Docker Compose operations (up/down/restart specific services)

**Safety rules**:
- On production: restrict to read-only operations (logs, status, health) unless explicitly deploying
- Container deletion requires user confirmation
- Docker socket access = root access — treat with appropriate security posture

---

### P1 Servers — Add After MVP Is Running

#### 6. grafana — Prometheus Metrics & Dashboard Inspection

| Field | Value |
|-------|-------|
| Package | `@grafana/mcp-grafana` |
| Install | `claude mcp add grafana --scope project -- npx -y @grafana/mcp-grafana` |
| Env | `GRAFANA_URL=http://localhost:3001`, `GRAFANA_API_KEY=<api-key>` |
| Purpose | Query Prometheus metrics via PromQL, inspect dashboards, manage alerts |
| When to install | After monitoring stack is generating meaningful data (Week 3+) |

**Why it's valuable for PolyBot**:
- All services expose `polybot_*` Prometheus metrics (counters, histograms, gauges)
- Performance debugging requires PromQL: `rate(polybot_orders_total{status="filled"}[5m])`, `histogram_quantile(0.95, polybot_execution_latency_seconds_bucket)`
- Alert rule inspection and management
- 6 pre-built Grafana dashboards referenced in `docs/09-infrastructure-spec.md`

**Key tools**: `search_dashboards`, `query_prometheus` (PromQL), `list_alert_rules`, `get_alert_rule`

---

#### 7. sequential-thinking — Complex Analysis & Reasoning

| Field | Value |
|-------|-------|
| Package | `@modelcontextprotocol/server-sequential-thinking` |
| Install | `claude mcp add sequential-thinking --scope project -- npx -y @modelcontextprotocol/server-sequential-thinking` |
| Purpose | Structured multi-step reasoning for complex analysis |
| When to install | During strategy development and risk parameter tuning |

**Why it's valuable for PolyBot**:
- Trading profit calculation: `profit = (1.0 - yes_ask - no_ask) * size - fees_both_sides - slippage`
- Risk parameter cascading: changing one threshold impacts 10 risk checks
- Wallet rebalancing: multi-constraint optimization across tiers, bots, and positions
- Circuit breaker tuning: failure rate thresholds, cool-down periods, recovery conditions

**When to use**: Invoke explicitly for strategy analysis and risk parameter reasoning. Not for routine coding.

---

### Already-Connected MCP Servers (Claude Code Plugins)

#### Playwright — Dashboard E2E Testing

**Verdict**: **HIGH VALUE — keep and actively leverage**

The Playwright MCP is already connected via Claude Code plugins. It directly supports the 5 E2E test scenarios defined in `docs/11-testing-strategy.md`:

| Test Scenario | Playwright Capability |
|--------------|----------------------|
| Paper trading end-to-end | Navigate, verify bot status cards, check P&L charts |
| Emergency stop end-to-end | Click emergency stop button, verify all bots show STOPPED |
| Dashboard navigation and data display | Navigate pages, verify data rendering, check SSE updates |
| Configuration change propagation | Change config via UI, verify updates reflect |
| Recovery after service restart | Verify dashboard reconnects SSE after service bounce |

**Integration with skills**: Used by `/deploy` (smoke test), `/e2e-test` (full test suites).

#### Supabase — Not for PolyBot

**Verdict**: Keep for Apex documentation agent, but **do NOT use for PolyBot development**.

PolyBot uses self-hosted PostgreSQL 16 + TimescaleDB. Supabase does not support TimescaleDB. Using Supabase as a dev database would create environment parity issues (hypertables, continuous aggregates, compression policies don't exist in Supabase).

---

### Servers NOT Recommended

| MCP Server | Reason to Skip |
|------------|----------------|
| Filesystem MCP | Fully redundant with Claude Code built-in Read/Write/Edit/Glob/Grep |
| GitHub MCP | `gh` CLI via Bash covers all needs for solo/small team |
| Generic WebSocket MCP | Polymarket WebSocket is handled internally by market data service |
| FastAPI MCP | Context7 already provides FastAPI documentation lookup |

---

## Context Window Budget

Each MCP server adds tool definitions to the context window.

| MCP Server | Est. Tools | Est. Tokens |
|------------|-----------|-------------|
| local-rag | 6 | ~800 |
| context7 | 2 | ~400 |
| postgres | ~10 | ~1,500 |
| redis | ~15 | ~2,000 |
| docker | ~8 | ~1,200 |
| **Phase 1 Total (5 servers)** | **~41** | **~5,900** |
| grafana | ~12 | ~1,800 |
| sequential-thinking | 1 | ~300 |
| **Phase 2 Total (7 servers)** | **~54** | **~8,000** |

At ~8,000 tokens for the full configuration, this is ~4% of a 200K context window — acceptable.

**Note**: The Supabase MCP plugin adds ~3,000+ tokens from 30+ tools. If context becomes tight during long sessions, Supabase is the first candidate for removal since it's not used for PolyBot.

---

## Slash Commands Specification

### P0 Commands — Must-Have for MVP Development

#### `/new-bot` — Scaffold New Bot Module

```
Usage: /new-bot {strategy_name}
Priority: P0
MCP Dependencies: None
File: docs/.claude/commands/new-bot.md → .claude/commands/new-bot.md
```

**What it does**:
1. Reads `docs/04-technical-specification.md` → BaseBot interface section for the contract
2. Creates directory: `src/bots/{strategy_name}/`
3. Creates `__init__.py` (empty)
4. Creates `bot.py` with class `{StrategyName}Bot(BaseBot)` implementing all 10 lifecycle methods
5. Creates `strategy.py` with a placeholder strategy class
6. Creates `models.py` with strategy-specific Pydantic models
7. Creates `config/bots/{strategy_name}-default.yaml` with `paper_trading: true`, `graduated: false`
8. Creates unit test: `tests/unit/test_{strategy_name}_strategy.py`
9. Uses `src/bots/binary_arbitrage/` as the reference implementation for structure
10. Runs `ruff check` and `mypy` on generated code

---

#### `/check-risk` — Audit Risk Configuration

```
Usage: /check-risk
Priority: P0
MCP Dependencies: postgres (ledger verification), redis (circuit breaker state)
File: docs/.claude/commands/check-risk.md → .claude/commands/check-risk.md
```

**What it does**:
1. Reads `docs/04-technical-specification.md` → Service 4: Risk Manager for the 10-step pipeline spec
2. Reads `config/risk.yaml` — validates all values within documented safe ranges
3. Reads all `config/bots/*.yaml` — verifies per-bot risk params don't exceed global limits
4. Validates virtual balances don't exceed wallet tier capacity
5. Verifies circuit breaker thresholds configured for every enabled bot
6. Cross-checks drawdown limits (step 9) and per-trade max loss (step 10)
7. If postgres MCP available: verifies software ledger balances match config
8. If redis MCP available: inspects live circuit breaker state
9. Produces structured report with PASS/WARN/FAIL per check

**Report format**:
```
RISK AUDIT REPORT
═════════════════
[PASS] Global risk limits within safe ranges
[PASS] Per-bot limits do not exceed global
[WARN] Bot "market_maker" has no max_drawdown_pct set (defaults to unlimited)
[FAIL] Bot "arb_v2" virtual_balance (5000) exceeds Alpha wallet capacity (4000)
[PASS] Circuit breaker thresholds configured for all 3 enabled bots
[PASS] Drawdown protection limits set for all bots
[INFO] Live circuit breaker state: all CLOSED
```

---

#### `/test-strategy` — Run Strategy Unit Tests

```
Usage: /test-strategy {bot_name}
Priority: P0
MCP Dependencies: None
File: docs/.claude/commands/test-strategy.md → .claude/commands/test-strategy.md
```

**What it does**:
1. Reads `docs/11-testing-strategy.md` → Critical Unit Test Suites for coverage expectations
2. Runs: `pytest tests/unit/test_{bot_name}_strategy.py -v --tb=long`
3. If tests pass, also runs: `pytest tests/integration/ -k {bot_name} -v`
4. Reports coverage: `pytest --cov=src/bots/{bot_name} --cov-report=term-missing`
5. Compares coverage against 70% target (from doc 11)
6. Formats results as structured report with pass/fail summary

---

#### `/deploy` — Build and Deploy

```
Usage: /deploy
Priority: P0
MCP Dependencies: docker (container management), playwright (smoke test)
File: docs/.claude/commands/deploy.md → .claude/commands/deploy.md
```

**What it does**:
1. Reads `docs/09-infrastructure-spec.md` → Deployment Workflow
2. Pre-flight checks:
   - `make test` (all tests pass)
   - `cd frontend && npm run build` (frontend compiles)
   - `ruff check src/` and `mypy src/` (lint/typecheck pass)
3. Build: `docker compose build`
4. Show config diff since last deploy: `git diff HEAD~1 config/`
5. **MANDATORY CHECKPOINT**: Present summary, require user confirmation
6. If confirmed: `docker compose up -d`
7. Post-deploy verification:
   - Tail logs for 30 seconds, check for errors
   - Verify all container health checks pass (docker MCP)
   - Verify Prometheus is scraping all targets
8. Smoke test: Use Playwright MCP to verify dashboard loads and shows bot status
9. If issues found: present rollback instructions

---

#### `/debug-bot` — Comprehensive Bot Diagnostic

```
Usage: /debug-bot {bot_id_or_name}
Priority: P0
MCP Dependencies: postgres, redis, docker (all P0 — will be available)
File: docs/.claude/commands/debug-bot.md → .claude/commands/debug-bot.md
```

**What it does** — 11-step diagnostic:

| Step | Check | Tool |
|------|-------|------|
| 1 | Bot status in orchestrator | Redis: `GET bot_status:{bot_id}` |
| 2 | Container health | Docker: container status + health check |
| 3 | Recent signals (last 20) | PostgreSQL: `SELECT * FROM signal WHERE bot_id = X ORDER BY created_at DESC LIMIT 20` |
| 4 | Recent fills (last 20) | PostgreSQL: `SELECT * FROM fill WHERE bot_id = X ORDER BY created_at DESC LIMIT 20` |
| 5 | Risk rejections (last 10) | PostgreSQL: `SELECT * FROM risk_event WHERE bot_id = X AND action = 'REJECT' ORDER BY created_at DESC LIMIT 10` |
| 6 | Circuit breaker state | Redis: `GET circuit_breaker:{bot_id}` |
| 7 | Market data stream health | Redis: `XINFO STREAM market_data:{token_id}` for subscribed markets |
| 8 | Consumer group lag | Redis: `XINFO GROUPS market_data:{token_id}` |
| 9 | Wallet balance | Redis: `HGETALL wallet_balance:{wallet_id}` |
| 10 | Prometheus metrics | Grafana MCP (if available): execution latency, signal rate, fill rate |
| 11 | Recent logs | Docker: container logs with bot_id filter |

**Output**: Structured diagnostic report with findings per step, severity levels, and suggested remediation actions.

This is the **single most impactful skill** — it replaces manually checking 6+ systems when a bot misbehaves.

---

### P1 Commands — High Value, Add After MVP

#### `/e2e-test` — Dashboard E2E Tests

```
Usage: /e2e-test [suite_name]
Priority: P1
MCP Dependencies: playwright (already connected)
File: docs/.claude/commands/e2e-test.md → .claude/commands/e2e-test.md
Suites: dashboard, emergency-stop, config-change, all (default)
```

**What it does**:
1. Reads `docs/11-testing-strategy.md` → E2E test specifications
2. Verifies dashboard is accessible (navigate to dashboard URL)
3. Runs specified test suite:
   - **dashboard**: Login, verify bot cards render, check charts load, verify SSE updates arrive
   - **emergency-stop**: Trigger emergency stop via UI, verify all bots show STOPPED, verify confirmation dialog
   - **config-change**: Change a bot parameter via dashboard API, verify UI reflects the change
4. Takes screenshots at key checkpoints
5. Reports pass/fail with screenshots for failures

---

#### `/wallet-audit` — Financial Reconciliation

```
Usage: /wallet-audit
Priority: P1
MCP Dependencies: postgres, redis
File: docs/.claude/commands/wallet-audit.md → .claude/commands/wallet-audit.md
```

**What it does**:
1. Reads `docs/04-technical-specification.md` → Service 5: Wallet Manager for ledger design
2. PostgreSQL: `SUM(virtual_balance)` per wallet vs `wallet.total_balance`
3. PostgreSQL: `SUM(pending_orders)` per bot vs `available_balance`
4. Redis: cached `wallet_balance:{wallet_id}` vs database values
5. Verify tier allocations match `config/wallets.yaml` (Vault ~70%, Alpha ~25%, Sweep ~5%)
6. Check for orphaned ledger entries (bot deleted but balance remaining)
7. (Phase 2+) Query on-chain balance via Polygon RPC and compare to database

**Report format**:
```
WALLET RECONCILIATION REPORT
═════════════════════════════
Wallet: alpha-1 (0xABC...DEF)
  [OK]   Virtual balances sum: $4,850.00 / Total: $5,000.00 (available: $150.00)
  [OK]   Redis cache matches database
  [WARN] Tier allocation 26.3% (target: 25%) — drift 1.3%

Wallet: vault-1 (0x123...456)
  [OK]   Virtual balances sum: $13,200.00 / Total: $14,000.00
  [OK]   Tier allocation 70.0% (target: 70%)

Overall: 2 OK, 1 WARN, 0 CRITICAL
```

---

#### `/market-scan` — Polymarket Opportunity Scanner

```
Usage: /market-scan [--min-profit 0.005] [--min-liquidity 1000]
Priority: P1
MCP Dependencies: context7 (for py-clob-client API reference)
File: docs/.claude/commands/market-scan.md → .claude/commands/market-scan.md
```

**What it does**:
1. Reads `docs/04-technical-specification.md` → Binary Arbitrage Strategy for the algorithm
2. Queries Gamma API (`https://gamma-api.polymarket.com/markets`) for active binary markets
3. For each binary pair: calculates `spread = yes_ask + no_ask - 1.0`
4. Factors in fee rates (from fee endpoint or config)
5. Calculates net profit per unit after fees
6. Ranks opportunities by expected profit
7. Flags: market liquidity, time to resolution, 24h volume

**Output**:
```
POLYMARKET BINARY ARBITRAGE SCAN
════════════════════════════════
Found 3 opportunities above $0.005 net profit/unit:

| # | Market | Yes Ask | No Ask | Spread | Fees | Net/Unit | Liquidity | Resolves |
|---|--------|---------|--------|--------|------|----------|-----------|----------|
| 1 | Will X win? | $0.52 | $0.45 | $0.030 | $0.004 | $0.026 | $45,000 | 2026-03-15 |
| 2 | Will Y happen? | $0.61 | $0.37 | $0.020 | $0.004 | $0.016 | $22,000 | 2026-04-01 |
| 3 | Will Z pass? | $0.48 | $0.50 | $0.020 | $0.004 | $0.016 | $12,000 | 2026-03-20 |
```

---

#### `/perf-profile` — Async Service Performance Profiler

```
Usage: /perf-profile {service_name}
Priority: P1
MCP Dependencies: grafana (if available), docker, postgres
File: docs/.claude/commands/perf-profile.md → .claude/commands/perf-profile.md
Services: market_data, orchestrator, execution, risk, wallet, dashboard
```

**What it does**:
1. Reads `docs/05-development-guidelines.md` → Performance Budget for latency targets
2. If Grafana MCP available, queries Prometheus:
   - `polybot_{service}_request_duration_seconds` (p50, p95, p99)
   - `polybot_{service}_active_tasks` (concurrent async tasks)
   - `polybot_{service}_errors_total` (error rate)
   - `polybot_redis_command_duration_seconds` (Redis latency)
   - `polybot_db_query_duration_seconds` (DB query latency)
3. Compares against performance budget targets from doc 05
4. If Grafana unavailable, checks container resource usage (Docker MCP)
5. Checks PostgreSQL slow query log if available
6. Identifies bottlenecks and suggests optimizations

---

### P2 Commands — Nice to Have

#### `/backtest` — Historical Strategy Backtesting

```
Usage: /backtest {bot_name} [--days 7] [--market "market_slug"]
Priority: P2
MCP Dependencies: postgres, sequential-thinking
File: docs/.claude/commands/backtest.md → .claude/commands/backtest.md
When to use: After historical data has accumulated (Week 3+)
```

**What it does**:
1. Reads the bot's `strategy.py` to understand the algorithm
2. Queries TimescaleDB for historical data:
   - `order_book_snapshot` hypertable for the time period
   - `fill` table for actual execution data (if available)
3. Runs signal generation logic against historical snapshots
4. Simulates P&L accounting for: fee rates, slippage estimation, rate limits (3,500/10s per wallet), position size limits
5. Uses Sequential Thinking MCP for step-by-step profit/loss calculations
6. Compares with actual performance if bot was live during the period

**Output**: Backtest report with total P&L, win rate, max drawdown, trade-by-trade log, and risk check simulation.

---

## Phase-Gated Rollout

### Phase 1: MVP Development (Weeks 1-6)

**Install**:
- 3 P0 MCP servers: postgres, redis, docker
- (local-rag and context7 already installed)

**Create**:
- 5 P0 slash commands: `/new-bot`, `/check-risk`, `/test-strategy`, `/deploy`, `/debug-bot`

**Total MCP servers**: 5 (existing 2 + new 3)

### Phase 2: Growth & Optimization (Weeks 7-12)

**Install**:
- Grafana MCP (monitoring data now meaningful)
- Sequential Thinking MCP (strategy analysis)

**Create**:
- 4 P1 slash commands: `/e2e-test`, `/wallet-audit`, `/market-scan`, `/perf-profile`

**Total MCP servers**: 7

### Phase 3: Advanced (Weeks 13+)

**Create**:
- `/backtest` command (historical data available)

**Total MCP servers**: 7 (no new servers)

---

## MCP Configuration File

The full configuration is maintained in `docs/mcp-config.json` (copy to `.mcp.json` in project root during setup).

See `docs/13-claude-code-setup.md` for installation and verification instructions.

---

## Cross-References

- **Agent roles**: `docs/07-agents-md.md` → AGENTS.md, sub-agent configs
- **Testing strategy**: `docs/11-testing-strategy.md` → E2E test scenarios, coverage targets
- **Infrastructure**: `docs/09-infrastructure-spec.md` → Docker Compose, deployment workflow, monitoring
- **Technical spec**: `docs/04-technical-specification.md` → Service designs, data model, risk pipeline
- **Development setup**: `docs/13-claude-code-setup.md` → MCP installation steps, verification
- **Project governance**: `docs/12-project-governance.md` → Deployment approval, change management

---

## Future Considerations

1. **Wallet Manager sub-agent**: Currently folded into execution-agent. As wallet logic grows (Phase 2 multi-EOA, Phase 3 cross-chain), consider splitting into a dedicated `docs/.claude/agents/wallet-agent.md`.
2. **AI/ML agent**: Phase 6 introduces ML ensemble strategies. Will need a dedicated sub-agent for `src/bots/ml_ensemble/` with access to model training tools.
3. **Rust sidecar MCP**: Phase 3 adds a Rust sidecar for low-latency signing. May need a dedicated MCP or enhanced Docker MCP integration for Rust container debugging.
