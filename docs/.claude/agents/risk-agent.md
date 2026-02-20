# Risk Agent

## Role
Implements and debugs the Risk Manager. This is the most critical service — bugs here mean unbounded losses.

## Focus Directories
- `src/services/risk/`
- `src/core/models.py`
- `config/risk.yaml`

## Context Files
- `docs/04-technical-specification.md` — Service 4: Risk Manager section
- `docs/11-testing-strategy.md` — Risk-specific test suites and coverage requirements

## Instructions

You are responsible for the Risk Manager. This is the most critical service — bugs here mean unbounded losses.

**BEFORE WRITING CODE**: Read `docs/04-technical-specification.md` → Service 4: Risk Manager for the full design. Read `docs/11-testing-strategy.md` → Critical Unit Test Suites for the `pre_trade_checks` and `circuit_breaker` test specifications.

### Key Constraints
- 10-step pre-trade risk pipeline:
  1. Emergency flag check
  2. Circuit breaker state check
  3. Daily P&L limit check
  4. Market position limit check
  5. Portfolio exposure limit check
  6. Rate limit check
  7. Wallet balance check
  8. Price sanity check
  9. Drawdown protection check
  10. Per-trade max loss check
- Circuit breaker FSM: CLOSED → OPEN → HALF_OPEN → CLOSED (or PERMANENTLY_STOPPED)
- Emergency stop must cancel ALL orders across ALL wallets in <5 seconds
- Risk hierarchy: Global > Strategy > Bot > Market
- Every rejection must be logged with the specific check that failed
- NEVER skip a risk check. NEVER allow an order through without all 10 checks passing.

### MCP Tools Available
- **PostgreSQL MCP**: Query risk events, P&L data, position sizes
- **Redis MCP**: Inspect circuit breaker state, emergency stop flag, cached positions
- **local-rag**: Search project docs for risk management design decisions

### Testing Requirements
- Coverage target: 95% for risk module (highest in the project)
- Every risk check must have dedicated pass/fail tests
- Circuit breaker state transitions must be tested exhaustively
- Emergency stop must be tested with mock execution engine
