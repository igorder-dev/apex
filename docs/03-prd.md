# Product Requirements Document: PolyBot Platform

## Overview

| Field | Value |
|-------|-------|
| Product | PolyBot — Automated Trading Platform for Polymarket |
| Author | Apex (PM/Architect/DevOps) |
| Status | Draft |
| Last Updated | 2026-02-15 |
| Target Release | Phase 1 MVP — Week 6 |
| Team Scale | Solo developer → small team (2–5) |
| Platform | Self-hosted VPS (Docker Compose) |

---

## Goals & Non-Goals

### Goals

1. **G1**: Build a modular, pluggable bot framework where each trading strategy is an independent module implementing a standard interface
2. **G2**: Provide an orchestration layer that manages bot lifecycles (start/stop/pause/resume/emergency stop), health monitoring, and centralized control
3. **G3**: Deliver an admin dashboard with real-time bot status, P&L tracking, position monitoring, risk indicators, and bot management controls
4. **G4**: Implement a risk management core with per-bot circuit breakers, daily loss caps, position limits, and emergency shutdown
5. **G5**: Deploy a wallet management system with risk-tier isolation (Vault/Alpha/Sweep), software ledger for per-bot P&L attribution, and automated profit sweeping
6. **G6**: Ship one profitable MVP strategy (Binary Arbitrage) that validates the entire infrastructure end-to-end
7. **G7**: Achieve >99% system uptime on a single VPS with full observability (metrics, logging, alerting)

### Non-Goals

1. **NG1**: Multi-tenant SaaS — this is a self-hosted platform for a single operator, not a multi-user service
2. **NG2**: Mobile app — the dashboard is a responsive web app, not a native mobile application
3. **NG3**: Code generation — the platform does not write trading strategies; it provides the runtime, execution, and management layer
4. **NG4**: Bypassing geoblocking — the platform enforces compliance; it does not circumvent Polymarket's jurisdiction restrictions
5. **NG5**: Guaranteed profitability — the platform provides infrastructure and risk controls, not profit guarantees
6. **NG6**: High-frequency trading (MVP) — sub-100ms execution is deferred to Phase 3 (Rust sidecar); MVP operates at 1–2 second execution latency
7. **NG7**: Multi-exchange support (MVP) — Kalshi and other platforms are deferred to Phase 6; MVP targets Polymarket only

---

## User Personas

### Persona 1: The Operator

- **Role**: Platform owner who deploys, monitors, and manages the trading system
- **Goals**: Maximize risk-adjusted returns across multiple strategies; maintain system health; manage capital allocation
- **Pain Points**: No unified view across bots; manual monitoring is exhausting and error-prone; missed opportunities while offline
- **Technical Proficiency**: Can manage a VPS, run Docker, edit YAML configs; may or may not write Python
- **Daily Workflow**: Check dashboard morning/evening; review P&L and alerts; adjust bot configs; investigate anomalies

### Persona 2: The Strategy Developer

- **Role**: Develops and tunes trading strategy modules (bots)
- **Goals**: Rapidly prototype new strategies; iterate on parameters; validate via paper trading before going live
- **Pain Points**: 80% of time spent on infrastructure instead of strategy; no standard interface to build against; no backtest framework
- **Technical Proficiency**: Strong Python developer; understands market microstructure and probability
- **Daily Workflow**: Write strategy code; run paper trades; analyze results; tune parameters; deploy to live

### Persona 3: The Risk Manager (same person, different hat)

- **Role**: Monitors risk exposure and ensures the system doesn't exceed loss boundaries
- **Goals**: Prevent catastrophic losses; ensure capital is allocated correctly across strategies; get alerted immediately on risk events
- **Pain Points**: No circuit breakers in existing scripts; no portfolio-level risk view; manual emergency shutdown is too slow
- **Technical Proficiency**: Understands position sizing, drawdown, and risk/reward ratios
- **Daily Workflow**: Review risk dashboard; check circuit breaker states; verify wallet balances and allocations

---

## User Stories & Requirements

### Epic 1: Bot Framework

| ID | User Story | Acceptance Criteria | Priority | Notes |
|----|------------|-------------------|----------|-------|
| US-101 | As a Strategy Developer, I want a standard bot interface (BaseBot ABC) so I can build strategies that plug into the platform without writing infrastructure code | Given a new Python class implementing BaseBot, When I place it in `src/bots/{name}/bot.py` and add a YAML config, Then the orchestrator discovers and loads it at startup | P0 | Core interface — must be stable from Day 1 |
| US-102 | As a Strategy Developer, I want each bot to receive a BotContext with injected services (market data, execution, risk) so I don't manage connections myself | Given a bot's `on_init(ctx)` is called, When I access `ctx.market_data`, `ctx.execution`, `ctx.risk`, Then they are fully initialized and connected | P0 | Dependency injection pattern |
| US-103 | As a Strategy Developer, I want lifecycle hooks (on_init, on_start, on_pause, on_resume, on_stop, on_emergency_stop) so my bot responds correctly to orchestrator commands | Given the orchestrator sends a "pause" command, When my bot's `on_pause()` is called, Then all my open orders are cancelled and no new signals are generated | P0 | All 7 hooks required |
| US-104 | As a Strategy Developer, I want an `on_market_data(token_id, data)` callback so I receive real-time order book updates for my subscribed markets | Given I subscribe to token_id "X" during on_init, When the order book for X updates, Then on_market_data is called within 500ms of the WebSocket message | P0 | Hot path — performance critical |
| US-105 | As a Strategy Developer, I want an `on_fill(fill)` callback so I'm notified when my orders are filled | Given I placed a limit order, When it fills on the CLOB, Then on_fill is called with fill price, size, side, and order ID | P0 | |
| US-106 | As a Strategy Developer, I want to report custom KPI metrics via `get_metrics()` so the dashboard displays my strategy-specific data | Given my bot computes a custom metric "arb_edge_bps", When get_metrics() is called, Then it returns {"arb_edge_bps": 42.5, ...} and the dashboard renders it | P1 | Extensible metrics |
| US-107 | As a Strategy Developer, I want paper trading mode so my bot generates signals without placing real orders | Given paper_trading=true in my bot config, When my bot emits a Signal, Then the Execution Engine logs it but does not submit to Polymarket | P0 | Essential for validation |
| US-108 | As a Strategy Developer, I want YAML-based bot configuration with Pydantic validation so misconfigurations are caught at startup | Given an invalid config (e.g., min_edge_bps: -5), When the orchestrator loads the config, Then it raises a ValidationError with a clear message and the bot does not start | P0 | Fail-fast on bad config |

### Epic 2: Orchestration

| ID | User Story | Acceptance Criteria | Priority | Notes |
|----|------------|-------------------|----------|-------|
| US-201 | As an Operator, I want to start/stop/pause/resume individual bots from the dashboard so I can manage them without SSH | Given the dashboard shows bot "arb-btc-01" as running, When I click "Pause", Then the bot's on_pause() is called, its state changes to PAUSED, and the dashboard reflects the change within 2 seconds | P0 | |
| US-202 | As an Operator, I want the orchestrator to auto-restart bots that crash (up to 3 retries) so transient failures don't require manual intervention | Given a bot crashes during execution, When the orchestrator detects the failure, Then it restarts the bot (max 3 attempts, with 30s backoff), and if all retries fail, marks the bot as ERROR and sends an alert | P0 | |
| US-203 | As an Operator, I want health checks every 10 seconds so I know immediately if a bot becomes unresponsive | Given a bot fails to respond to 3 consecutive health checks (30s total), When the orchestrator detects this, Then the bot is stopped, an alert is sent, and the dashboard shows ERROR state | P0 | |
| US-204 | As an Operator, I want a single "Emergency Stop All" button that cancels all orders and halts all bots within 5 seconds | Given any state, When I press Emergency Stop, Then all open orders are cancelled via `cancel_all()` per wallet, all bots transition to STOPPED, and the system enters read-only mode | P0 | Must work even if dashboard is slow |
| US-205 | As an Operator, I want the orchestrator to enforce single-instance per bot config so I can't accidentally run duplicate bots on the same market | Given bot config "arb-btc-01" is already running, When I attempt to start another instance with the same config, Then the orchestrator rejects it with "Bot already running" | P0 | Prevents double-execution |
| US-206 | As an Operator, I want to hot-reload bot configuration without restarting the bot so I can tune parameters during market hours | Given bot "mm-politics-01" is running, When I update its YAML config and click "Reload" in the dashboard, Then the bot receives the updated config via on_config_update() and applies new parameters without stopping | P1 | Phase 1 stretch goal |

### Epic 3: Market Data

| ID | User Story | Acceptance Criteria | Priority | Notes |
|----|------------|-------------------|----------|-------|
| US-301 | As a Bot, I want real-time order book snapshots from Polymarket's CLOB WebSocket so I can compute trading signals | Given I subscribe to a market's token_id, When the order book changes, Then I receive an OrderBookSnapshot with bids, asks, spread, midpoint, and timestamp within 500ms | P0 | |
| US-302 | As the Platform, I want automatic WebSocket reconnection with exponential backoff so market data isn't lost during transient disconnections | Given the WebSocket disconnects, When reconnection is attempted, Then it retries with backoff (1s, 2s, 4s, 8s, max 30s) up to 10 times, then alerts and falls back to REST polling | P0 | |
| US-303 | As the Platform, I want WebSocket connection pooling (max 500 instruments per connection) so I can monitor more than 500 markets | Given 800 active subscriptions, When connections are managed, Then 2 WebSocket connections are maintained, each subscribing to ≤500 instruments | P0 | Polymarket hard limit |
| US-304 | As a Bot, I want market discovery via the Gamma API so new markets are automatically identified as they launch | Given the Gamma API poller runs every 60 seconds, When a new market appears that matches my configured filters, Then it's added to the available markets list and I can subscribe to it | P1 | |
| US-305 | As the Platform, I want cached order books in Redis so any service can access the latest book without subscribing to WebSocket directly | Given the Market Data Service receives an order book update, When it publishes to Redis Stream and updates the cache, Then any service calling `redis.hgetall("orderbook:{token_id}")` gets the latest snapshot | P0 | |

### Epic 4: Execution

| ID | User Story | Acceptance Criteria | Priority | Notes |
|----|------------|-------------------|----------|-------|
| US-401 | As a Bot, I want to place FOK, GTC, GTD, and IOC orders through the Execution Engine so I don't interact with the CLOB API directly | Given I submit a Signal with order_type="FOK", When the Execution Engine processes it, Then a FOK order is placed via `py-clob-client` with correct signing, fee-rate, and decimal precision | P0 | |
| US-402 | As the Platform, I want client-side rate limiting (token bucket: 3,500 burst/10s per wallet) so we never hit Polymarket's 429 throttle | Given 3,500 orders have been sent in the last 10 seconds, When another order is submitted, Then it's queued and delayed until the rate limit window resets | P0 | Per-wallet rate limits |
| US-403 | As the Platform, I want batch order support (up to 15 orders per call) so multi-leg strategies can submit atomically | Given a bot emits 2 signals for YES and NO legs, When the Execution Engine batches them, Then both are submitted in a single `post_orders()` call | P0 | Critical for arb strategies |
| US-404 | As the Platform, I want dynamic fee-rate caching (refresh every 30s per token) so orders include correct fees for fee-enabled markets | Given a 15-minute BTC market with taker fee 2.5%, When a bot places a taker order, Then the order includes the correct fee_rate_bps and the Execution Engine has refreshed the rate within the last 30 seconds | P0 | |
| US-405 | As the Platform, I want an order state machine (PENDING → SUBMITTED → LIVE → MATCHED/CANCELLED/EXPIRED) so I always know each order's status | Given an order is submitted, When it transitions through states, Then each transition is logged, timestamped, and the dashboard reflects the current state | P0 | |
| US-406 | As the Platform, I want order reconciliation every 30 seconds so internal state matches Polymarket's actual state | Given internal state shows 5 LIVE orders, When reconciliation runs, Then it queries `/orders` and corrects any discrepancies (e.g., externally cancelled orders) | P0 | Drift detection |
| US-407 | As the Platform, I want wallet-aware order routing so orders from different bots are routed to the correct wallet/API key based on tier assignment | Given bot "arb-btc-01" is assigned to Vault wallet, When it submits an order, Then the Execution Engine uses Vault wallet's API key and the order is attributed to the Vault proxy wallet | P0 | Wallet architecture |

### Epic 5: Risk Management

| ID | User Story | Acceptance Criteria | Priority | Notes |
|----|------------|-------------------|----------|-------|
| US-501 | As the Risk Manager, I want per-bot daily loss caps so a malfunctioning bot can't drain the account | Given bot "arb-btc-01" has max_daily_loss_usdc=200, When its cumulative daily loss reaches $200, Then all its open orders are cancelled, it transitions to STOPPED, and an alert is sent | P0 | |
| US-502 | As the Risk Manager, I want per-market position limits so no single market concentrates too much exposure | Given max_position_per_market_usdc=500, When a bot attempts an order that would push position to $550, Then the order is rejected by the pre-trade risk check | P0 | |
| US-503 | As the Risk Manager, I want a circuit breaker (CLOSED → OPEN → HALF-OPEN) that triggers after N consecutive losses | Given circuit_breaker.consecutive_losses_trigger=5, When a bot has 5 consecutive losing trades, Then the circuit breaker opens, all orders are cancelled, and a 5-minute cooldown begins. After cooldown, HALF-OPEN allows 1 test trade. | P0 | |
| US-504 | As the Risk Manager, I want a pre-trade risk pipeline (8 checks) that every order must pass before submission | Given an order is submitted, When it reaches the Execution Engine, Then it passes through: (1) emergency stop flag, (2) circuit breaker state, (3) daily P&L cap, (4) market position limit, (5) portfolio exposure, (6) rate limit capacity, (7) wallet balance sufficiency, (8) price sanity check (0.01–0.99) | P0 | All 8 checks |
| US-505 | As the Risk Manager, I want global portfolio exposure limits so total deployed capital never exceeds the configured maximum | Given max_portfolio_value_usdc=50000, When all positions across all bots total $48K and a new $3K order arrives, Then it's rejected with "Would exceed portfolio limit" | P0 | |
| US-506 | As the Risk Manager, I want automatic emergency shutdown if 10 consecutive API errors occur so a platform outage doesn't cause orphaned orders | Given 10 consecutive API failures (timeouts, 5xx, connection errors), When the threshold is breached, Then emergency shutdown is triggered system-wide | P0 | |

### Epic 6: Wallet Management

| ID | User Story | Acceptance Criteria | Priority | Notes |
|----|------------|-------------------|----------|-------|
| US-601 | As an Operator, I want to configure multiple Polymarket wallets (separate EOAs) organized by risk tier so capital is isolated | Given I add 3 EOA private keys in `.env` and assign them to tiers in `config/wallets.yaml`, When the platform starts, Then the Wallet Manager initializes each wallet, derives API keys, and verifies proxy wallet deployment | P0 | Phase 1: 1 wallet; Phase 2+: multi-wallet |
| US-602 | As an Operator, I want to assign bots to wallet tiers so low-risk and high-risk strategies use separate capital pools | Given bot config contains `wallet_tier: vault`, When the bot is loaded, Then the orchestrator routes its orders through the Vault wallet's API key and the software ledger attributes trades to this bot within the Vault wallet | P0 | |
| US-603 | As the Platform, I want a software ledger that tracks per-bot positions and P&L within shared wallets so I know exactly which bot owns what | Given Bot A and Bot B both trade in the Vault wallet, When Bot A buys 100 YES tokens for Market X, Then the ledger records: wallet=Vault, bot=A, market=X, side=YES, size=100, and per-bot P&L is calculated independently | P0 | |
| US-604 | As the Platform, I want per-bot virtual balance allocation within each wallet so one bot can't overspend another bot's budget | Given Bot A has virtual_balance=2000 USDC within the Vault wallet, When Bot A attempts a $2,100 order, Then the pre-trade risk check rejects it with "Exceeds virtual balance allocation" even if the wallet has sufficient total balance | P0 | Software-enforced, not on-chain |
| US-605 | As an Operator, I want automated profit sweeping that transfers realized profits to the Sweep wallet on a configurable schedule | Given sweep_schedule=daily and sweep_threshold_usdc=100, When daily profit in the Vault wallet exceeds $100, Then the excess is automatically transferred to the Sweep wallet via Polygon USDC transfer | P1 | Phase 2 |
| US-606 | As an Operator, I want inter-wallet rebalancing when tier allocations drift beyond ±10% of target | Given target allocation is 70% Vault / 25% Alpha / 5% Sweep, When actual is 80% / 15% / 5%, Then the Rebalancer transfers USDC from Vault to Alpha to restore target (within configurable tolerance) | P1 | Phase 2 |
| US-607 | As an Operator, I want per-wallet balance monitoring with alerts when balance drops below a threshold | Given alert_threshold_usdc=500 for the Vault wallet, When the Vault balance drops below $500, Then a Telegram alert is sent and the dashboard shows a warning indicator | P1 | |
| US-608 | As an Operator, I want to see per-wallet balances, allocations, and P&L breakdown in the dashboard | Given 2 active wallets, When I open the Wallet view in the dashboard, Then I see each wallet's USDC balance, conditional token positions, per-bot P&L attribution, and allocation vs. target | P1 | |

### Epic 7: Dashboard

| ID | User Story | Acceptance Criteria | Priority | Notes |
|----|------------|-------------------|----------|-------|
| US-701 | As an Operator, I want a System Overview page showing portfolio value, daily P&L, active bots count, and recent alerts | Given the dashboard loads, When I'm on the Overview page, Then I see total portfolio value, daily P&L ($ and %), active/paused/stopped bot counts, and the last 20 alert events — all updating in real-time | P0 | |
| US-702 | As an Operator, I want a Bot Management table with status, P&L, and action buttons per bot | Given 5 bots are configured, When I view the Bot Management page, Then I see a table with columns: Name, Strategy, Status (badge), Wallet Tier, Today P&L, Positions, Last Heartbeat, Actions (Start/Pause/Stop) | P0 | |
| US-703 | As an Operator, I want a Bot Detail view with P&L chart, open positions, trade history, and signal log | Given I click into bot "arb-btc-01", When the detail view loads, Then I see: intraday P&L line chart, positions table (market, side, size, entry price, current price, unrealized P&L), trade history (filterable by date), and signal log (signal, action taken, result) | P1 | |
| US-704 | As an Operator, I want a Market Monitor page showing tracked markets with spreads, volumes, and arb opportunities | Given 50 markets are monitored, When I view Market Monitor, Then I see a table with: market name, YES price, NO price, YES+NO sum, spread, 24h volume, and a highlight indicator when YES+NO deviates from 1.00 by >50bps | P1 | |
| US-705 | As an Operator, I want a Risk Dashboard showing exposure breakdown, circuit breaker states, and daily loss tracking | Given the system is running, When I view Risk Dashboard, Then I see: portfolio exposure by wallet/bot/market (bar charts), circuit breaker status per bot (green/yellow/red), daily loss vs. cap (progress bars), and risk event log | P1 | |
| US-706 | As an Operator, I want a Settings page for global risk parameters, wallet configuration, and alert settings | Given I navigate to Settings, When I update max_daily_loss_usdc from 1000 to 800, Then the change is validated, saved, and applied to the Risk Manager without restart | P1 | |
| US-707 | As an Operator, I want real-time data streaming via SSE so dashboard data updates automatically without polling | Given I'm viewing the Overview page, When a trade fills, Then the portfolio value and P&L update within 1 second without page refresh | P0 | SSE for streaming |

### Epic 8: Binary Arbitrage Bot (MVP Strategy)

| ID | User Story | Acceptance Criteria | Priority | Notes |
|----|------------|-------------------|----------|-------|
| US-801 | As a Trader, I want the bot to continuously scan binary markets for full-set parity violations (YES_ask + NO_ask < 1.00) | Given the bot is monitoring 50 binary markets, When any market's YES_ask + NO_ask drops below 1.00 - min_edge_bps, Then a LONG_ARB signal is generated within 2 seconds | P0 | |
| US-802 | As a Trader, I want the bot to execute both legs (buy YES + buy NO) via FOK orders to minimize partial fill risk | Given a LONG_ARB signal, When the bot executes, Then it submits FOK BUY orders for both YES and NO via the batch order API in a single call | P0 | |
| US-803 | As a Trader, I want partial fill handling so a single-leg fill doesn't leave me with unhedged directional exposure | Given one FOK leg fills but the other is rejected, When partial fill is detected, Then the bot either: (a) sets a limit sell to close the filled leg at breakeven, or (b) queues the unfilled leg as a GTC order, configurable via `partial_fill_strategy` in bot config | P0 | |
| US-804 | As a Trader, I want the bot to compute edge net of all costs (fees, gas estimate, slippage buffer) so it only trades when truly profitable | Given YES_ask=0.48, NO_ask=0.50, fee=0bps, gas_estimate=$0.01, slippage_buffer=20bps, When edge is calculated, Then net_edge = 1.00 - 0.48 - 0.50 - 0.00 - 0.01 - 0.002 = $0.008; trade executes only if net_edge > min_edge_bps | P0 | |
| US-805 | As a Trader, I want the bot to also detect SHORT_ARB (YES_bid + NO_bid > 1.00 + costs) for completeness | Given YES_bid=0.53, NO_bid=0.49, When the sum (1.02) exceeds 1.00 + costs, Then a SHORT_ARB signal is generated and the bot executes split + sell both sides | P0 | |
| US-806 | As a Trader, I want configurable sizing modes: fixed USDC, percentage of bankroll, or Kelly criterion | Given `size_mode: fixed, fixed_size_usdc: 100`, When an arb opportunity is detected, Then the bot trades 100 USDC per leg (or available depth, whichever is smaller) | P0 | |
| US-807 | As a Trader, I want the bot to respect the maximum concurrent arbitrages limit so capital isn't over-committed | Given max_concurrent_arbs=5 and 5 arbs are currently open, When a new opportunity appears, Then it's logged but not executed until a slot frees up | P1 | |

### Epic 9: Observability

| ID | User Story | Acceptance Criteria | Priority | Notes |
|----|------------|-------------------|----------|-------|
| US-901 | As an Operator, I want structured JSON logs across all services with correlation IDs for request tracing | Given a trade flows from Market Data → Bot → Execution, When I search logs by correlation_id, Then I see the complete flow across all services | P0 | structlog |
| US-902 | As an Operator, I want Prometheus metrics for orders placed, fill rates, P&L, latency, WebSocket reconnects, and error rates | Given Prometheus scrapes every 15 seconds, When I open Grafana, Then I see time-series charts for all key trading and system metrics | P1 | |
| US-903 | As an Operator, I want Telegram alerts for: errors, circuit breaks, daily P&L summary, emergency shutdown, and low wallet balance | Given a circuit breaker triggers, When the event occurs, Then I receive a Telegram message within 10 seconds with bot name, reason, and current P&L | P1 | |
| US-904 | As an Operator, I want an audit log in PostgreSQL recording every order, fill, risk event, and config change with timestamps | Given any state-changing action, When it occurs, Then a row is inserted into the audit_log table with: timestamp, actor (bot/operator/system), action, details, and wallet | P0 | Compliance + debugging |
| US-905 | As an Operator, I want to enable debug mode per bot to log every decision step (market data evaluation, signal computation, risk checks, order decisions, fill processing) for post-trade analysis | Given `debug_mode: true` in bot config, When the bot processes market data, Then every decision step is logged at DEBUG level to a per-bot log file (`logs/{bot_id}_debug.jsonl`) | P1 | Algorithm improvement |
| US-906 | As an Operator, I want drawdown protection that pauses a bot if its P&L drops more than X% from peak equity | Given `max_drawdown_pct: 15.0` in bot config, When the bot's equity drops 15% from its peak, Then the bot is paused and I receive a Telegram alert | P0 | Capital protection |
| US-907 | As an Operator, I want per-trade loss limits that reject any trade where the maximum possible loss exceeds a threshold | Given `max_loss_per_trade_usdc: 50` in bot config, When a signal would produce a trade with >$50 max loss, Then the order is rejected with reason logged | P0 | Capital protection |
| US-908 | As an Operator, I want new bots to always start in paper trading mode and require my explicit approval to go live | Given `require_paper_graduation: true` globally, When a new bot is created, Then it starts in paper mode and refuses to trade live until I set `graduated: true` | P0 | Safety |
| US-909 | As an Operator, I want a system-wide paper mode toggle that forces ALL bots into paper mode regardless of individual settings | Given `system_paper_mode: true` in global config, When any bot tries to trade, Then all orders are simulated regardless of per-bot `paper_trading` setting | P1 | Safety |

---

## Functional Requirements

### FR-1: Bot Runtime

- **FR-1.1**: The platform SHALL dynamically discover and load bot modules from `src/bots/{strategy_name}/bot.py` at startup
- **FR-1.2**: The platform SHALL validate that each bot class implements all required `BaseBot` methods before instantiation
- **FR-1.3**: The platform SHALL inject a `BotContext` into each bot's `on_init()` with references to Market Data, Execution, Risk, Config, Logger, and Metrics services
- **FR-1.4**: The platform SHALL maintain an independent async event loop (or task group) per bot within the Orchestrator process
- **FR-1.5**: The platform SHALL support adding new bot modules without restarting unrelated bots (hot plugin discovery)

### FR-2: Market Data

- **FR-2.1**: The Market Data Service SHALL maintain persistent WebSocket connections to `wss://ws-subscriptions-clob.polymarket.com/ws/`
- **FR-2.2**: The service SHALL enforce the 500-instrument-per-connection limit by spawning additional connections
- **FR-2.3**: The service SHALL normalize all incoming data to internal Pydantic models before publishing to Redis Streams
- **FR-2.4**: The service SHALL cache the latest order book per token_id in Redis with <100ms staleness
- **FR-2.5**: The service SHALL poll the Gamma API every 60 seconds for market discovery and metadata refresh

### FR-3: Execution

- **FR-3.1**: The Execution Engine SHALL wrap `py-clob-client` and expose a `submit_order(signal, wallet_tier)` method
- **FR-3.2**: The engine SHALL enforce client-side rate limits: 3,500 orders per 10-second window per wallet, using a token bucket algorithm
- **FR-3.3**: The engine SHALL support batch order submission (up to 15 orders per `post_orders()` call)
- **FR-3.4**: The engine SHALL query and cache fee rates per token_id, refreshing every 30 seconds for fee-enabled markets
- **FR-3.5**: The engine SHALL validate FOK order decimal precision (sell amounts ≤2 decimals, taker amounts ≤4 decimals, size × price ≤2 decimals)
- **FR-3.6**: The engine SHALL maintain an order state machine per order with transitions logged and timestamped
- **FR-3.7**: The engine SHALL reconcile internal order state against Polymarket `/orders` endpoint every 30 seconds
- **FR-3.8**: The engine SHALL route orders to the correct wallet API key based on the submitting bot's tier assignment

### FR-4: Risk Management

- **FR-4.1**: The Risk Manager SHALL execute the 10-step pre-trade risk pipeline before every order submission
- **FR-4.2**: The Risk Manager SHALL maintain per-bot P&L tracking (realized + unrealized) with daily reset at 00:00 UTC
- **FR-4.3**: The Risk Manager SHALL implement circuit breaker FSM per bot with configurable trigger thresholds and cooldown periods
- **FR-4.4**: The Risk Manager SHALL enforce portfolio-wide exposure limits across all wallets
- **FR-4.5**: The Risk Manager SHALL execute emergency shutdown within 5 seconds of trigger, cancelling all orders across all wallets
- **FR-4.6**: The Risk Manager SHALL log all risk events (rejections, circuit breaks, limit breaches) to the audit log

### FR-5: Wallet Management

- **FR-5.1**: The Wallet Manager SHALL support 1–N wallet configurations, each backed by a unique EOA private key
- **FR-5.2**: The Wallet Manager SHALL derive and cache API credentials per wallet using `create_or_derive_api_creds()`
- **FR-5.3**: The Wallet Manager SHALL maintain a software ledger mapping: order_id → bot_id → wallet_tier
- **FR-5.4**: The Wallet Manager SHALL enforce per-bot virtual balance limits within shared wallets
- **FR-5.5**: The Wallet Manager SHALL implement a per-wallet mutex to prevent concurrent order race conditions from the same wallet
- **FR-5.6**: The Wallet Manager SHALL support automated USDC transfers between wallets for rebalancing (Phase 2)
- **FR-5.7**: The Wallet Manager SHALL support automated profit sweeping to the Sweep wallet (Phase 2)

### FR-6: Dashboard

- **FR-6.1**: The Dashboard SHALL be a React SPA served as static files by the FastAPI backend
- **FR-6.2**: The Dashboard SHALL stream real-time data via Server-Sent Events (SSE) at 1-second intervals
- **FR-6.3**: The Dashboard SHALL provide REST endpoints for CRUD operations on bot configs, risk settings, and wallet configs
- **FR-6.4**: The Dashboard SHALL require authentication (API key or session) before granting access
- **FR-6.5**: The Dashboard SHALL provide an Emergency Stop button that is always visible and functional regardless of page state

---

## Non-Functional Requirements

| Category | Requirement | Target | Measurement |
|----------|-------------|--------|-------------|
| **Performance** | Market data → bot signal latency | <500ms p95 | Timestamp tracking (WS receive → on_market_data call) |
| **Performance** | Signal → order submission latency | <2s p95 (MVP) | Timestamp tracking (signal emit → API call) |
| **Performance** | Dashboard page load | <3s initial, <100ms updates | Browser DevTools / Lighthouse |
| **Performance** | Emergency shutdown completion | <5s | End-to-end test (button press → all orders cancelled) |
| **Availability** | System uptime | >99% over rolling 7 days | Prometheus uptime metric |
| **Availability** | WebSocket reconnection | <30s to restore data feed after disconnect | Market Data Service logs |
| **Scalability** | Concurrent bots | ≥10 bots simultaneously | Load test on target VPS |
| **Scalability** | Monitored markets | ≥500 markets (1 WS connection) | Market Data Service metrics |
| **Scalability** | Historical data retention | ≥12 months of trade data | TimescaleDB compression policy |
| **Reliability** | Order reconciliation drift | <0.1% orders in inconsistent state | Reconciliation loop metrics |
| **Reliability** | Zero orphaned orders after shutdown | 100% of orders cancelled on shutdown | Emergency shutdown test |
| **Security** | Private key encryption | AES-256 at rest | Encrypted .env or Docker secrets |
| **Security** | Dashboard authentication | Required for all endpoints | Auth middleware |
| **Security** | No secrets in logs | 0 occurrences of keys/passwords in log output | Log audit |
| **Observability** | Log coverage | 100% of state-changing operations logged | Code review |
| **Observability** | Metric scrape interval | 15 seconds | Prometheus config |
| **Observability** | Alert delivery latency | <10s from event to Telegram notification | Alert pipeline test |
| **Accessibility** | Dashboard WCAG | Level AA | Lighthouse audit |
| **Resource** | Total VPS RAM usage | <12GB for full stack | Docker stats |
| **Resource** | Total VPS CPU usage | <80% sustained for 10 bots | Docker stats |

---

## Out of Scope / Future Considerations

| Feature | Phase | Rationale for Deferral |
|---------|-------|----------------------|
| Rust execution sidecar (<100ms signing) | Phase 3 | Only needed for latency arbitrage; Python is adequate for MVP strategies |
| Market Making bot | Phase 2 | Requires volatility calculator, backtest framework, and Avellaneda-Stoikov implementation |
| Copy Trading bot | Phase 4 | Requires on-chain wallet monitoring infrastructure |
| NegRisk / Multi-Outcome arbitrage | Phase 5 | Requires conversion graph builder and multi-leg execution |
| AI/ML ensemble models | Phase 6 | Requires RAG pipeline, LLM integration, and calibration framework |
| Cross-platform arbitrage (Kalshi) | Phase 6 | Requires Kalshi API integration |
| Multi-tenant support | Not planned | Platform is single-operator by design |
| Mobile native app | Not planned | Responsive web dashboard is sufficient |
| Automated strategy backtesting | Phase 2 | Phase 1 uses paper trading for validation |
| Hot-reload bot code (not just config) | Phase 3 | Requires dynamic module reloading with state preservation |

---

## Open Questions

| # | Question | Impact | Status |
|---|----------|--------|--------|
| OQ-1 | What is the specific VPS jurisdiction? Polymarket geoblocks several countries including Poland. | If VPS IP is in a blocked jurisdiction, order placement fails. | **Needs user input** |
| OQ-2 | Will we use Polymarket WebSocket premium tier ($99/month) from Day 1, or start with REST polling and upgrade later? | Premium unlocks WebSocket; without it, bots are limited to REST polling (100 req/min). | **Recommend premium from Day 1** — REST polling is inadequate for arb detection |
| OQ-3 | Initial trading capital for live Phase 1? ($500 minimum recommended, $2K ideal) | Determines position sizing and expected daily P&L. | **Needs user input** |
| OQ-4 | Is Polymarket US (regulated, sports-only) or Polymarket Global (crypto wallets, all markets) the target? | Polymarket US requires KYC and has different market types. | **Assume Global** — broader market access, more strategies viable |
| OQ-5 | Should the Sweep wallet support automated withdrawal to an external wallet (e.g., Ethereum mainnet, CEX), or just accumulate on Polygon? | Affects withdrawal bridge implementation. | **Phase 2 decision** — accumulate on Polygon for MVP |

---

## Appendix: Cross-Reference to Other Documents

| Reference | Document | Section |
|-----------|----------|---------|
| Technology stack rationale | [04-technical-specification.md](./04-technical-specification.md) | Technology Stack + ADRs |
| Wallet architecture details | [04-technical-specification.md](./04-technical-specification.md) | Wallet Manager Service |
| Security threat model | [08-security-spec.md](./08-security-spec.md) | Threat Model |
| Infrastructure deployment | [09-infrastructure-spec.md](./09-infrastructure-spec.md) | Docker Compose |
| API endpoint specifications | [10-api-specification.md](./10-api-specification.md) | Full API spec |
| Testing acceptance criteria | [11-testing-strategy.md](./11-testing-strategy.md) | Test plans per epic |
| Phase timeline and milestones | [02-product-roadmap.md](./02-product-roadmap.md) | Phase details |
| Market research and risk assessment | [01-product-research.md](./01-product-research.md) | Risk Assessment |
