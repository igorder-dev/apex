# Development Guidelines: PolyBot Platform

## Repository Structure

```
polybot/
├── docker-compose.yml                 # Production service definitions
├── docker-compose.dev.yml             # Development overrides (hot reload, debug ports)
├── .env.example                       # Template — NEVER commit real .env
├── Makefile                           # Common commands: up, down, logs, test, backup
├── README.md                          # Project overview, quickstart
├── CLAUDE.md                          # AI coding agent instructions
├── AGENTS.md                          # Multi-agent coordination config
├── pyproject.toml                     # Python project config (dependencies, tools)
├── alembic.ini                        # Database migration config
│
├── config/
│   ├── risk.yaml                      # Global risk parameters
│   ├── wallets.yaml                   # Wallet tier configuration
│   ├── markets.yaml                   # Market watchlist and filters
│   ├── prometheus.yml                 # Prometheus scrape targets
│   ├── grafana/
│   │   └── dashboards/                # Pre-configured Grafana dashboards (JSON)
│   └── bots/
│       ├── binary-arb-default.yaml    # Per-bot configuration files
│       └── ...
│
├── src/
│   ├── core/
│   │   ├── __init__.py
│   │   ├── bot_interface.py           # BaseBot ABC + BotContext + Signal + HealthStatus
│   │   ├── models.py                  # Shared Pydantic models (OrderBook, Fill, Position, etc.)
│   │   ├── config.py                  # YAML config loader with Pydantic validation
│   │   ├── exceptions.py              # Custom exception hierarchy
│   │   └── enums.py                   # Shared enums (BotState, OrderStatus, Side, etc.)
│   │
│   ├── services/
│   │   ├── market_data/
│   │   │   ├── __init__.py
│   │   │   ├── service.py             # MarketDataService entrypoint
│   │   │   ├── websocket_manager.py   # WS connection pool, reconnection, 500-instrument limit
│   │   │   ├── gamma_client.py        # Gamma API wrapper (market discovery)
│   │   │   ├── normalizer.py          # Raw payload → Pydantic model conversion
│   │   │   ├── publisher.py           # Redis Streams + Cache publisher
│   │   │   └── Dockerfile
│   │   │
│   │   ├── orchestrator/
│   │   │   ├── __init__.py
│   │   │   ├── service.py             # Orchestrator entrypoint
│   │   │   ├── bot_loader.py          # Dynamic module discovery + loading
│   │   │   ├── lifecycle.py           # Bot state machine (CREATED→RUNNING→STOPPED)
│   │   │   ├── health_checker.py      # 10s health check loop
│   │   │   ├── commands.py            # Redis Pub/Sub command listener
│   │   │   └── Dockerfile
│   │   │
│   │   ├── execution/
│   │   │   ├── __init__.py
│   │   │   ├── service.py             # ExecutionEngine entrypoint
│   │   │   ├── clob_client_wrapper.py # py-clob-client wrapper + wallet-aware routing
│   │   │   ├── order_state_machine.py # Order lifecycle tracking
│   │   │   ├── reconciliation.py      # 30s order state reconciliation
│   │   │   ├── rate_limiter.py        # Token bucket (3,500/10s per wallet)
│   │   │   ├── fee_cache.py           # Dynamic fee rate cache (30s refresh)
│   │   │   └── Dockerfile
│   │   │
│   │   ├── risk/
│   │   │   ├── __init__.py
│   │   │   ├── service.py             # RiskManager entrypoint
│   │   │   ├── circuit_breaker.py     # Circuit breaker FSM (CLOSED/OPEN/HALF_OPEN)
│   │   │   ├── position_tracker.py    # Real-time position tracking per bot/market
│   │   │   ├── pnl_calculator.py      # Realized + unrealized P&L
│   │   │   ├── pre_trade_checks.py    # 10-step risk pipeline
│   │   │   └── Dockerfile
│   │   │
│   │   ├── wallet/
│   │   │   ├── __init__.py
│   │   │   ├── service.py             # WalletManager entrypoint
│   │   │   ├── initializer.py         # Multi-EOA init, API key derivation, proxy detection
│   │   │   ├── ledger.py              # Software ledger (per-bot P&L attribution)
│   │   │   ├── balance_monitor.py     # 30s balance polling, low-balance alerts
│   │   │   ├── rebalancer.py          # Inter-wallet USDC transfers (Phase 2)
│   │   │   ├── sweeper.py             # Profit sweep to cold wallet (Phase 2)
│   │   │   └── Dockerfile
│   │   │
│   │   └── dashboard/
│   │       ├── __init__.py
│   │       ├── app.py                 # FastAPI application factory
│   │       ├── auth.py                # API key / session auth middleware
│   │       ├── routes/
│   │       │   ├── bots.py            # Bot CRUD + lifecycle endpoints
│   │       │   ├── trades.py          # Trade history
│   │       │   ├── positions.py       # Open positions
│   │       │   ├── risk.py            # Risk dashboard
│   │       │   ├── wallets.py         # Wallet management
│   │       │   ├── markets.py         # Market data
│   │       │   ├── system.py          # Health, emergency stop
│   │       │   └── settings.py        # Global settings
│   │       ├── sse.py                 # Server-Sent Events streaming
│   │       └── Dockerfile
│   │
│   ├── bots/
│   │   ├── binary_arbitrage/
│   │   │   ├── __init__.py
│   │   │   ├── bot.py                 # BinaryArbitrageBot(BaseBot)
│   │   │   ├── strategy.py            # Full-set parity scanner algorithm
│   │   │   └── models.py              # Strategy-specific models
│   │   └── ...                        # Future: market_making/, latency_arb/, etc.
│   │
│   └── shared/
│       ├── __init__.py
│       ├── redis_client.py            # Async Redis connection + Streams helpers
│       ├── db.py                      # SQLAlchemy async engine + session factory
│       ├── metrics.py                 # Prometheus metric definitions
│       ├── logging_config.py          # structlog JSON configuration
│       └── telegram.py                # Alert bot (send messages, daily summaries)
│
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── tailwind.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── components/
│   │   │   ├── ui/                    # shadcn/ui components
│   │   │   ├── layout/               # Shell, sidebar, header
│   │   │   ├── bots/                  # Bot-specific components
│   │   │   ├── charts/               # P&L charts, order book viz
│   │   │   └── risk/                  # Risk indicators, circuit breaker badges
│   │   ├── pages/
│   │   │   ├── Overview.tsx
│   │   │   ├── BotManagement.tsx
│   │   │   ├── BotDetail.tsx
│   │   │   ├── MarketMonitor.tsx
│   │   │   ├── RiskDashboard.tsx
│   │   │   └── Settings.tsx
│   │   ├── hooks/
│   │   │   ├── useSSE.ts             # Server-Sent Events connection hook
│   │   │   └── useBotControl.ts      # Bot action API calls
│   │   └── lib/
│   │       ├── api.ts                 # REST API client (axios/fetch wrapper)
│   │       └── types.ts              # TypeScript types matching Pydantic models
│   └── dist/                          # Production build output (served by FastAPI)
│
├── tests/
│   ├── conftest.py                    # Shared fixtures (test DB, Redis mock, bot context)
│   ├── unit/
│   │   ├── test_circuit_breaker.py
│   │   ├── test_rate_limiter.py
│   │   ├── test_pnl_calculator.py
│   │   ├── test_pre_trade_checks.py
│   │   ├── test_order_state_machine.py
│   │   ├── test_binary_arb_strategy.py
│   │   ├── test_software_ledger.py
│   │   └── ...
│   ├── integration/
│   │   ├── test_execution_flow.py     # Signal → risk → order → fill
│   │   ├── test_bot_lifecycle.py      # Start → pause → resume → stop
│   │   ├── test_wallet_routing.py     # Bot → wallet → API key routing
│   │   └── ...
│   ├── e2e/
│   │   ├── test_paper_trading.py      # Full pipeline in paper mode
│   │   ├── test_emergency_stop.py     # Emergency shutdown E2E
│   │   └── ...
│   └── fixtures/
│       ├── orderbook_snapshots.json   # Sample order book data
│       ├── fill_events.json           # Sample fill events
│       └── market_metadata.json       # Sample Gamma API responses
│
├── migrations/
│   ├── env.py                         # Alembic environment config
│   └── versions/                      # Migration files (auto-generated)
│
└── scripts/
    ├── setup-vps.sh                   # VPS provisioning (Docker, UFW, SSH)
    ├── backup.sh                      # PostgreSQL backup + offsite copy
    ├── restore.sh                     # Restore from backup
    └── generate-api-creds.py          # One-time Polymarket credential generation
```

---

## Development Environment Setup

### Prerequisites

- Python 3.11+
- Node.js 20+ (for frontend build)
- Docker 24+ and Docker Compose 2.x
- Git

### Setup from Clean Machine

```bash
# 1. Clone repository
git clone <repo-url> polybot && cd polybot

# 2. Create virtual environment
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
# .venv\Scripts\activate   # Windows

# 3. Install Python dependencies
pip install -e ".[dev]"

# 4. Copy environment template
cp .env.example .env
# Edit .env: add VAULT_PRIVATE_KEY, TELEGRAM_BOT_TOKEN, etc.

# 5. Start infrastructure (DB + Redis only)
docker compose -f docker-compose.dev.yml up -d postgres redis

# 6. Run database migrations
alembic upgrade head

# 7. Install frontend dependencies
cd frontend && npm install && cd ..

# 8. Run in development mode
make dev  # Starts all services with hot reload
```

### Development Mode vs Production

| Aspect | Development | Production |
|--------|-------------|------------|
| Python services | `uvicorn --reload` (auto-restart on code change) | `uvicorn` with gunicorn workers |
| Frontend | `npm run dev` (Vite dev server with HMR) | `npm run build` → static files served by FastAPI |
| Database | `docker compose -f docker-compose.dev.yml` | `docker compose` (production) |
| Logging | DEBUG level, console output | INFO level, JSON to Docker log driver |
| Paper trading | Default ON | Configurable per bot |
| Hot reload | Enabled | Disabled |

### Makefile Commands

```makefile
# Infrastructure
up:           docker compose up -d
down:         docker compose down
dev:          docker compose -f docker-compose.dev.yml up
logs:         docker compose logs -f --tail=100

# Database
migrate:      alembic upgrade head
migrate-new:  alembic revision --autogenerate -m "$(msg)"
db-shell:     docker compose exec postgres psql -U polybot

# Testing
test:         pytest tests/ -v
test-unit:    pytest tests/unit/ -v
test-int:     pytest tests/integration/ -v
test-cov:     pytest tests/ --cov=src --cov-report=html

# Frontend
fe-dev:       cd frontend && npm run dev
fe-build:     cd frontend && npm run build
fe-lint:      cd frontend && npm run lint

# Operations
backup:       ./scripts/backup.sh
restore:      ./scripts/restore.sh $(file)
api-creds:    python scripts/generate-api-creds.py
```

---

## Git Workflow

### Branching Strategy: GitHub Flow (Trunk-Based)

Single `main` branch as the source of truth. Feature branches for all changes. No develop/staging branches — simplicity for solo dev → small team.

```
main ─────────────────────────────────────────────────────
  │                     │                    │
  ├── feat/US-101-bot-interface             │
  │       └── PR → main ──────────┐         │
  │                                │         │
  ├── feat/US-301-websocket-manager         │
  │       └── PR → main ──────────┐         │
  │                                │         │
  ├── fix/arb-partial-fill-handling         │
  │       └── PR → main ───────────────────┐│
  │                                         ││
  main ◄────────────────────────────────────┘│
```

### Branch Naming

```
{type}/{ticket-id}-{short-description}

Types:
  feat/    — New feature or capability
  fix/     — Bug fix
  refactor/— Code restructure, no behavior change
  test/    — Test additions or fixes
  docs/    — Documentation only
  infra/   — Docker, CI/CD, deployment changes
  perf/    — Performance optimization

Examples:
  feat/US-101-bot-interface
  fix/arb-partial-fill-race-condition
  refactor/execution-engine-async
  infra/grafana-trading-dashboard
```

### Commit Format: Conventional Commits

```
{type}({scope}): {description}

[optional body]

[optional footer]
```

**Types**: `feat`, `fix`, `refactor`, `test`, `docs`, `ci`, `perf`, `chore`

**Scopes**: `core`, `market-data`, `orchestrator`, `execution`, `risk`, `wallet`, `dashboard`, `arb-bot`, `infra`, `state`

**Examples**:
```
feat(core): add BaseBot ABC with lifecycle hooks
fix(execution): handle FOK decimal precision for sell orders
refactor(risk): extract pre-trade checks into pipeline pattern
test(arb-bot): add unit tests for full-set parity scanner
docs(readme): add development environment setup instructions
ci(docker): add health checks to all service containers
perf(market-data): batch Redis XADD calls for order book updates
chore(state): update project state after US-101 BaseBot implementation
```

### STATE.md Update on Commit

Every code commit should be accompanied by an update to `STATE.md` (project root). This can be:
- **Combined**: STATE.md changes included in the same commit as the code (preferred)
- **Separate**: A follow-up `chore(state): ...` commit immediately after

See `docs/12-project-governance.md` → Agent Feature Delivery Protocol step 9 for the full protocol.

### PR Requirements

- **Title**: Follows commit format (squash-merged into main)
- **Description**: What changed, why, how to test
- **Checks**: All tests pass, linting clean
- **Review**: Self-review for solo dev; 1 reviewer for team of 2+
- **Squash merge**: Always squash into a single commit on main

---

## Code Standards

### Python (enforced by tooling, not documentation)

**Tool Configuration** (`pyproject.toml`):

```toml
[project]
name = "polybot"
requires-python = ">=3.11"

[tool.ruff]
target-version = "py311"
line-length = 100
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "A",    # flake8-builtins
    "SIM",  # flake8-simplify
    "TCH",  # flake8-type-checking
    "RUF",  # ruff-specific
]

[tool.ruff.isort]
known-first-party = ["src"]

[tool.mypy]
python_version = "3.11"
strict = true
plugins = ["pydantic.mypy", "sqlalchemy.ext.mypy.plugin"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
addopts = "-v --tb=short"
```

**Key Conventions** (not enforced by linter — team knowledge):

- All service entrypoints follow the same pattern: `async def main()` with signal handlers for graceful shutdown
- All database operations use the async SQLAlchemy session pattern (no sync sessions)
- All Redis operations use the async redis-py client
- Pydantic models are the contract between services — never pass raw dicts across service boundaries
- Type hints on all function signatures (enforced by mypy strict mode)

### TypeScript/React

**Tool Configuration**:

```json
// frontend/.eslintrc.json (simplified)
{
  "extends": ["@typescript-eslint/recommended", "plugin:react-hooks/recommended"],
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-unused-vars": "error"
  }
}
```

- Strict TypeScript (`strict: true` in tsconfig)
- Functional components only (no class components)
- shadcn/ui components used as-is; custom components in `components/`
- API types generated from Pydantic models (manual sync for MVP; codegen in Phase 2)

---

## Testing Requirements

| Type | Coverage Target | Framework | When to Run |
|------|----------------|-----------|-------------|
| Unit | 80% for `core/`, `risk/`, `execution/` | pytest + pytest-asyncio | Pre-commit, CI |
| Integration | Critical paths (signal → order → fill) | pytest + testcontainers | CI |
| E2E | Paper trading pipeline, emergency stop | pytest + Docker Compose | Pre-deploy |
| Frontend | Component rendering, API integration | Vitest + React Testing Library | CI |

### Critical Test Cases (must exist from Day 1)

1. **Circuit breaker**: transitions CLOSED→OPEN→HALF_OPEN→CLOSED with correct timing
2. **Pre-trade risk pipeline**: all 8 checks with pass/reject scenarios
3. **Rate limiter**: token bucket refill, depletion, and queueing behavior
4. **Order state machine**: all valid transitions, rejection of invalid transitions
5. **Binary arb strategy**: edge detection with realistic order book data, partial fill handling
6. **Software ledger**: credit/debit accuracy, virtual balance enforcement, P&L calculation
7. **Wallet routing**: correct API key selection based on bot→tier mapping
8. **Emergency stop**: all orders cancelled, all bots stopped, system enters read-only mode

### Test Fixtures

```python
# tests/conftest.py — key fixtures
@pytest.fixture
async def test_db():
    """Spin up a fresh PostgreSQL + TimescaleDB for integration tests."""
    # Uses testcontainers-python for ephemeral database

@pytest.fixture
async def redis_mock():
    """In-memory Redis mock for unit tests."""
    # Uses fakeredis-py with async support

@pytest.fixture
def sample_orderbook():
    """Realistic binary market order book with YES/NO pairs."""
    return OrderBookSnapshot(
        token_id="yes_token_123",
        bids=[(0.52, 500), (0.51, 1000), (0.50, 2000)],
        asks=[(0.53, 500), (0.54, 1000), (0.55, 2000)],
        timestamp=datetime.utcnow(),
    )

@pytest.fixture
def bot_context(redis_mock, test_db):
    """Fully wired BotContext for integration tests."""
    # Returns BotContext with mocked services
```

---

## CI/CD Pipeline

### Pipeline Stages

```mermaid
flowchart LR
    PUSH["git push"] --> LINT["Lint & Type Check<br/>ruff + mypy"]
    LINT --> TEST["Unit Tests<br/>pytest tests/unit/"]
    TEST --> INT["Integration Tests<br/>pytest tests/integration/<br/>(Docker services)"]
    INT --> BUILD["Docker Build<br/>All service images"]
    BUILD --> E2E["E2E Tests<br/>pytest tests/e2e/<br/>(Docker Compose)"]
    E2E --> UAT["User Validation<br/>(agent demos to user)"]
    UAT --> DEPLOY["Deploy<br/>(manual trigger)"]
```

### User Acceptance (UAT)

After automated tests pass, agents **must demo the feature to the user** before deploying. This is a mandatory human-in-the-loop gate. See `docs/12-project-governance.md` → Agent Feature Delivery Protocol for the full workflow.

**For bot/strategy changes**: Paper trading validation is required in addition to user acceptance. See `docs/11-testing-strategy.md` → Paper Trading Validation Checklist.

### GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install ruff mypy
      - run: ruff check src/
      - run: mypy src/

  test:
    runs-on: ubuntu-latest
    needs: lint
    services:
      redis:
        image: redis:7-alpine
        ports: [6379:6379]
      postgres:
        image: timescale/timescaledb:latest-pg16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: polybot_test
        ports: [5432:5432]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -e ".[dev]"
      - run: alembic upgrade head
        env: { DATABASE_URL: "postgresql+asyncpg://postgres:test@localhost/polybot_test" }
      - run: pytest tests/unit/ tests/integration/ -v --cov=src --cov-report=xml

  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20" }
      - run: cd frontend && npm ci && npm run lint && npm run build
```

### Deployment (Manual for MVP)

```bash
# On VPS:
cd /opt/polybot
git pull origin main
docker compose build
docker compose up -d
docker compose logs -f --tail=50  # Verify startup
```

Phase 2+ will add a `deploy` Makefile target and optional GitHub Actions deployment via SSH.

---

## Documentation Standards

| What | Where | Format |
|------|-------|--------|
| Architecture decisions | `docs/04-technical-specification.md` ADRs | ADR template |
| API endpoints | `docs/10-api-specification.md` | OpenAPI-compatible tables |
| Bot strategy algorithms | `src/bots/{name}/README.md` | Markdown with pseudocode |
| Config file schemas | Inline YAML comments + Pydantic model docstrings | Code-as-documentation |
| Runbooks (operations) | `docs/runbooks/` (Phase 2) | Step-by-step Markdown |
| Changelog | `CHANGELOG.md` | Keep a Changelog format |

**What NOT to document**:
- Code that is self-documenting via type hints and Pydantic models
- Style rules that ruff/mypy enforce
- Generic Python/React best practices

---

## Dependency Management

### Adding Dependencies

```bash
# Python: add to pyproject.toml [project.dependencies] or [project.optional-dependencies.dev]
# Then: pip install -e ".[dev]"

# Frontend: npm install <package>
```

**Rules**:
- Every new dependency must have a clear justification (don't add a library for something stdlib can do)
- Pin major versions in pyproject.toml (e.g., `fastapi>=0.115,<1.0`)
- Run `pip audit` and `npm audit` weekly to check for vulnerabilities
- No dependencies with fewer than 100 GitHub stars (unless official Polymarket SDK)

### Key Dependencies and Pinning Strategy

| Package | Pin Strategy | Rationale |
|---------|-------------|-----------|
| `py-clob-client` | Pin exact (`==0.34.5`) | Core trading SDK; breaking changes = trading failures |
| `fastapi` | Pin minor (`>=0.115,<0.120`) | Stable API, but minor versions can add breaking deprecations |
| `sqlalchemy` | Pin major (`>=2.0,<3.0`) | Async engine pattern is 2.0+; major version change would require rewrite |
| `redis` | Pin major (`>=5.0,<6.0`) | Streams API stable in 5.x |
| `pydantic` | Pin major (`>=2.0,<3.0`) | V2 is a different API from V1 |

### Security Scanning

```bash
# Python: pip-audit (run in CI)
pip-audit --strict

# Frontend: npm audit
cd frontend && npm audit --audit-level=moderate

# Docker: Trivy (run in CI)
trivy image polybot-execution:latest
```

---

## Performance Budget

### Core Web Vitals (Dashboard)

| Metric | Target | Tool |
|--------|--------|------|
| Largest Contentful Paint (LCP) | <2.5s | Lighthouse |
| First Input Delay (FID) | <100ms | Lighthouse |
| Cumulative Layout Shift (CLS) | <0.1 | Lighthouse |
| Bundle size (gzipped) | <500KB | Vite build stats |

### Trading Performance

| Metric | Target | Measurement |
|--------|--------|-------------|
| Market data → bot callback | <500ms p95 | Timestamp tracking |
| Signal → order submission | <2s p95 (MVP) | Timestamp tracking |
| Order reconciliation cycle | <5s | Reconciliation loop timer |
| Redis Stream publish latency | <10ms p95 | Redis metrics |
| PostgreSQL write latency | <50ms p95 | SQLAlchemy metrics |
| Dashboard SSE delivery | <100ms from event | SSE timestamp tracking |

### API Latency Budget

| Endpoint | Target p95 | Notes |
|----------|-----------|-------|
| `GET /api/bots` | <200ms | Cached in Redis |
| `GET /api/trades?limit=50` | <500ms | TimescaleDB query |
| `POST /api/bots/{id}/start` | <1s | Includes bot initialization |
| `POST /api/system/emergency-stop` | <2s | Must cancel all orders |
| `GET /api/stream/events` (SSE) | <100ms per event | Streaming, not request/response |

---

## Cross-References

| Topic | Document |
|-------|----------|
| Architecture diagrams, data model | [04-technical-specification.md](./04-technical-specification.md) |
| User stories these guidelines support | [03-prd.md](./03-prd.md) |
| Docker Compose configuration | [09-infrastructure-spec.md](./09-infrastructure-spec.md) |
| API endpoint specifications | [10-api-specification.md](./10-api-specification.md) |
| Full testing strategy | [11-testing-strategy.md](./11-testing-strategy.md) |
| Contribution process, PR review | [12-project-governance.md](./12-project-governance.md) |
