# /backtest — Historical Strategy Backtesting

When the user runs `/backtest $ARGUMENTS`, run the specified strategy against historical data.

Arguments:
- `{bot_name}` — required, the strategy to backtest
- `--days 7` — optional, number of days of history (default: 7)
- `--market "market_slug"` — optional, filter to specific market

## Prerequisites

- **PostgreSQL MCP** — required for historical data queries
- **Sequential Thinking MCP** (P1) — recommended for step-by-step P&L calculations
- Historical data must exist in TimescaleDB (order_book_snapshot, fill tables)

## Steps

1. **Read the strategy**: Read `src/bots/{bot_name}/strategy.py` to understand the signal generation algorithm, entry/exit conditions, and parameters.

2. **Read strategy config**: Read `config/bots/{bot_name}-default.yaml` for risk parameters, position sizing, and market subscriptions.

3. **Query historical data** (PostgreSQL MCP):
   ```sql
   -- Order book snapshots for the time period
   SELECT timestamp, token_id, best_bid, best_ask, bid_depth, ask_depth
   FROM order_book_snapshot
   WHERE token_id IN ('{token_ids}')
     AND timestamp >= NOW() - INTERVAL '{days} days'
   ORDER BY timestamp ASC
   ```

4. **Simulate signal generation**: Walk through historical snapshots chronologically. For each snapshot:
   - Apply the strategy's signal generation logic
   - Record generated signals with timestamps
   - Apply the 10-step risk pipeline checks (using config parameters)

5. **Simulate execution**: For each signal that passes risk checks:
   - Calculate execution price (use historical best bid/ask + slippage estimate)
   - Apply fee rates (historical or current)
   - Respect rate limits (3,500 orders/10s per wallet)
   - Respect position size limits from config

6. **Calculate P&L**: Use Sequential Thinking MCP (if available) for step-by-step calculations:
   - Per-trade P&L: `(exit_price - entry_price) * size - fees`
   - Running P&L with cumulative tracking
   - Maximum drawdown from peak equity
   - Win rate and average win/loss

7. **Compare with actual** (if bot was live during the period):
   ```sql
   SELECT * FROM fill WHERE bot_id = '{bot_id}'
     AND created_at >= NOW() - INTERVAL '{days} days'
   ORDER BY created_at
   ```
   Calculate divergence between simulated and actual results.

8. **Produce backtest report**:

```
BACKTEST REPORT: {bot_name}
═══════════════════════════
Period: {start_date} to {end_date} ({days} days)
Markets: {list}
Data points: {count} order book snapshots

PERFORMANCE SUMMARY
  Total P&L:          $125.40
  Win Rate:           68% (34/50 trades)
  Avg Win:            $8.20
  Avg Loss:           -$4.10
  Max Drawdown:       -$42.00 (3.2% of starting equity)
  Sharpe Ratio:       1.85 (annualized)
  Total Fees Paid:    $18.60

RISK SIMULATION
  Signals generated:  62
  Risk-rejected:      12 (19%)
    - wallet_balance: 5
    - position_limit: 4
    - rate_limit: 3
  Executed:           50

TRADE LOG (last 10)
  | # | Timestamp | Market | Side | Price | Size | P&L | Fees |
  |---|-----------|--------|------|-------|------|-----|------|
  | 50 | ... | ... | BUY | $0.52 | 100 | +$2.40 | $0.36 |
  ...

ACTUAL vs SIMULATED (if live data available)
  Simulated P&L:  $125.40
  Actual P&L:     $118.20
  Divergence:     +$7.20 (6.1%) — likely due to slippage estimation
```

## Notes
- This command requires historical data in TimescaleDB — won't work until the platform has been running
- Results are estimates: actual slippage, fill rates, and market impact will differ
- Use this for strategy validation and parameter tuning, not as a guarantee of future performance
- For markets that have resolved, the backtest is more accurate (final prices known)
