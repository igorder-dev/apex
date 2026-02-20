# /market-scan — Polymarket Opportunity Scanner

When the user runs `/market-scan $ARGUMENTS`, scan current Polymarket markets for trading opportunities.

Optional arguments:
- `--min-profit 0.005` — minimum net profit per unit after fees (default: $0.005)
- `--min-liquidity 1000` — minimum market liquidity in USDC (default: $1,000)
- `--strategy arbitrage` — strategy type: `arbitrage` (default), `spread`, `all`

## Steps

1. **Read the binary arbitrage algorithm**: Read `docs/04-technical-specification.md` → Binary Arbitrage Strategy section for the profit calculation formula and constraints.

2. **Fetch active markets**: Query the Gamma API for active binary markets:
   ```
   GET https://gamma-api.polymarket.com/markets?closed=false&active=true
   ```
   Use web fetch or Bash with curl.

3. **Filter binary markets**: Keep only markets with exactly 2 outcomes (YES/NO tokens).

4. **For each binary pair, calculate opportunity**:
   - Fetch order book data for both YES and NO tokens
   - Calculate spread: `spread = yes_ask + no_ask - 1.0`
   - If spread < 0 (prices sum to less than $1.00): arbitrage opportunity exists
   - Factor in fee rates (use current fee schedule or cached rates)
   - Calculate net profit per unit: `net = abs(spread) - fees_both_sides`
   - Estimate slippage based on order book depth

5. **Rank opportunities**: Sort by net profit per unit descending. Filter by minimum profit and liquidity thresholds.

6. **Enrich with context**: For top opportunities, note:
   - Market liquidity (total value locked)
   - 24-hour trading volume
   - Time to resolution
   - Any relevant market context

7. **Present results**:

```
POLYMARKET BINARY ARBITRAGE SCAN — {timestamp}
═══════════════════════════════════════════════
Filters: min profit > $0.005/unit, min liquidity > $1,000
Markets scanned: 142 binary markets
Opportunities found: 3

| # | Market | Yes Ask | No Ask | Spread | Fees | Net/Unit | Liq. | Resolves |
|---|--------|---------|--------|--------|------|----------|------|----------|
| 1 | Will X win? | $0.52 | $0.45 | $0.030 | $0.004 | $0.026 | $45K | Mar 15 |
| 2 | Will Y happen? | $0.61 | $0.37 | $0.020 | $0.004 | $0.016 | $22K | Apr 1 |
| 3 | Will Z pass? | $0.48 | $0.50 | $0.020 | $0.004 | $0.016 | $12K | Mar 20 |

NOTES
- Opportunity #1 has highest profit but resolves in 26 days (capital lock-up)
- Opportunity #2 has good liquidity for position sizing
- Fee rates based on current schedule; verify before trading

NO OPPORTUNITIES: If none found, report "No arbitrage opportunities above threshold."
```

## Notes
- This is a READ-ONLY scan — does not place any orders
- Use Context7 MCP for py-clob-client API reference if needed
- Market conditions change rapidly — results are a snapshot, not a recommendation
- For live trading decisions, always verify with the actual order book at execution time
