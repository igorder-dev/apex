# /wallet-audit — Financial Reconciliation

When the user runs `/wallet-audit`, perform a comprehensive wallet balance reconciliation.

## Prerequisites

- **PostgreSQL MCP** — required for ledger queries
- **Redis MCP** — required for cache comparison

If either MCP is unavailable, skip those checks and note them as `[SKIP]`.

## Steps

1. **Read wallet architecture**: Read `docs/04-technical-specification.md` → Service 5: Wallet Manager for the 3-tier wallet model and software ledger design.

2. **Read wallet config**: Read `config/wallets.yaml` for tier allocations and wallet definitions.

3. **Virtual balance reconciliation** (PostgreSQL):
   ```sql
   -- Per wallet: sum of virtual balances should not exceed total
   SELECT w.wallet_id, w.total_balance,
          COALESCE(SUM(le.amount), 0) as allocated,
          w.total_balance - COALESCE(SUM(le.amount), 0) as available
   FROM wallet w
   LEFT JOIN ledger_entry le ON le.wallet_id = w.wallet_id AND le.status = 'active'
   GROUP BY w.wallet_id, w.total_balance
   ```

4. **Pending orders check** (PostgreSQL):
   ```sql
   -- Per bot: pending order value should not exceed virtual balance
   SELECT bot_id, SUM(price * size) as pending_value
   FROM "order" WHERE status = 'PENDING'
   GROUP BY bot_id
   ```
   Compare against each bot's virtual balance.

5. **Cache consistency** (Redis):
   - For each wallet: compare `HGETALL wallet_balance:{wallet_id}` with PostgreSQL values
   - Flag any drift > $1.00 as WARN, > $10.00 as CRITICAL

6. **Tier allocation check**:
   - Calculate actual allocation percentage per tier
   - Compare against config targets (Vault ~70%, Alpha ~25%, Sweep ~5%)
   - Flag drift > 5% as WARN

7. **Orphan check** (PostgreSQL):
   ```sql
   -- Ledger entries for bots that no longer exist
   SELECT le.bot_id, SUM(le.amount) as orphaned_balance
   FROM ledger_entry le
   LEFT JOIN bot_config bc ON le.bot_id = bc.bot_id
   WHERE bc.bot_id IS NULL AND le.status = 'active'
   GROUP BY le.bot_id
   ```

8. **Produce reconciliation report**:

```
WALLET RECONCILIATION REPORT — {timestamp}
═══════════════════════════════════════════

TIER: VAULT (target: 70%)
  vault-1 (0xABC...DEF):
    [OK] Total: $14,000.00 | Allocated: $13,200.00 | Available: $800.00
    [OK] Redis cache: matches (drift: $0.00)
    [OK] Tier allocation: 70.0% (target: 70%)

TIER: ALPHA (target: 25%)
  alpha-1 (0x123...456):
    [OK] Total: $5,000.00 | Allocated: $4,850.00 | Available: $150.00
    [OK] Redis cache: matches (drift: $0.12)
    [WARN] Tier allocation: 26.3% (target: 25%) — drift 1.3%

TIER: SWEEP (target: 5%)
  sweep-1 (0x789...012):
    [OK] Total: $1,000.00 | Allocated: $400.00 | Available: $600.00

PENDING ORDERS
  binary_arbitrage_01: $120.00 pending (virtual balance: $2,000.00) [OK]
  market_maker_01: $890.00 pending (virtual balance: $1,000.00) [WARN — 89% utilized]

ORPHAN CHECK
  [OK] No orphaned ledger entries found

SUMMARY
  Total assets: $20,000.00
  Total allocated: $18,450.00
  Total available: $1,550.00
  Issues: 0 CRITICAL, 1 WARN, 0 FAIL
```

## Notes
- Run this before and after any wallet rebalancing operation
- Recommended frequency: daily or after any manual fund movements
- In Phase 2+, add on-chain balance verification via Polygon RPC
