# /check-risk — Audit Risk Configuration

When the user runs `/check-risk`, perform a comprehensive risk configuration audit.

## Steps

1. **Read the risk pipeline spec**: Read `docs/04-technical-specification.md` → Service 4: Risk Manager. Note the 10-step pre-trade pipeline:
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

2. **Read global risk config**: Read `config/risk.yaml`. Validate all values are present and within safe ranges:
   - `max_daily_loss_usdc` > 0
   - `max_portfolio_exposure_pct` between 0 and 100
   - `emergency_stop_loss_usdc` > `max_daily_loss_usdc`
   - Circuit breaker thresholds are reasonable (failure_threshold > 0, recovery_time > 0)

3. **Read all bot configs**: Glob `config/bots/*.yaml`. For each enabled bot:
   - Verify per-bot risk params don't exceed global limits
   - Verify `max_position_usdc` > 0
   - Verify `max_daily_loss_usdc` > 0
   - Verify `max_drawdown_pct` is set (warn if missing — defaults to unlimited)
   - Verify `max_loss_per_trade_usdc` is set

4. **Validate wallet capacity**: For each bot, check that `virtual_balance` (or `max_position_usdc`) doesn't exceed the assigned wallet tier's total capacity.

5. **Verify circuit breakers**: Ensure every enabled bot has circuit breaker thresholds configured (either globally or per-bot).

6. **MCP-enhanced checks** (if available):
   - **PostgreSQL MCP**: Query `SELECT bot_id, SUM(virtual_balance) FROM ledger_entry GROUP BY bot_id` and compare to config values
   - **Redis MCP**: Check live circuit breaker state for all bots (`GET circuit_breaker:{bot_id}`)

7. **Produce report**: Format results with severity levels:
   - `[PASS]` — Check passed
   - `[WARN]` — Non-critical issue (e.g., missing optional parameter)
   - `[FAIL]` — Critical issue that could cause unbounded losses
   - `[INFO]` — Informational (e.g., live system state)

## Output Format

```
RISK AUDIT REPORT — {timestamp}
═══════════════════════════════

GLOBAL CONFIGURATION
  [PASS] max_daily_loss_usdc: $500 (within safe range)
  [PASS] max_portfolio_exposure_pct: 30% (within safe range)
  [PASS] emergency_stop_loss_usdc: $1000 > max_daily_loss ($500)

PER-BOT CHECKS
  binary_arbitrage_01:
    [PASS] Position limit $100 <= global $500
    [PASS] Drawdown protection: 15.0%
    [PASS] Per-trade max loss: $10
  market_maker_01:
    [WARN] max_drawdown_pct not set — defaults to unlimited
    [FAIL] virtual_balance $5000 exceeds Alpha wallet capacity $4000

CIRCUIT BREAKERS
  [PASS] All 3 enabled bots have thresholds configured
  [INFO] Live state: all CLOSED (healthy)

SUMMARY: 8 PASS, 1 WARN, 1 FAIL
```
