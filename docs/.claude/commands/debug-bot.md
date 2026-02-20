# /debug-bot — Comprehensive Bot Diagnostic

When the user runs `/debug-bot $ARGUMENTS`, perform an 11-step diagnostic across all system layers.

Extract `{bot_id}` or `{bot_name}` from the arguments. If a name is given, resolve to bot_id using config files.

## Prerequisites

This command works best with all 3 P0 MCP servers installed:
- **PostgreSQL MCP** — for signal, fill, and risk event queries
- **Redis MCP** — for stream health, circuit breaker state, cache inspection
- **Docker MCP** — for container health and logs

If an MCP is unavailable, skip those steps and note them as `[SKIP]` in the report.

## Diagnostic Steps

### Step 1: Bot Status
- **Redis**: `GET bot_status:{bot_id}` — expected values: RUNNING, PAUSED, STOPPED, ERROR, INITIALIZING
- If status is ERROR or missing, flag immediately

### Step 2: Container Health
- **Docker MCP**: Check orchestrator container status and health
- Look for OOM kills, restart loops, or unhealthy state

### Step 3: Recent Signals (last 20)
- **PostgreSQL MCP**:
  ```sql
  SELECT signal_id, market_id, signal_type, value, confidence, created_at
  FROM signal WHERE bot_id = '{bot_id}'
  ORDER BY created_at DESC LIMIT 20
  ```
- Check: Are signals being generated? Is the rate reasonable? Any gaps?

### Step 4: Recent Fills (last 20)
- **PostgreSQL MCP**:
  ```sql
  SELECT fill_id, order_id, market_id, side, price, size, fee, created_at
  FROM fill WHERE bot_id = '{bot_id}'
  ORDER BY created_at DESC LIMIT 20
  ```
- Check: Are orders getting filled? What's the fill rate?

### Step 5: Risk Rejections (last 10)
- **PostgreSQL MCP**:
  ```sql
  SELECT risk_event_id, check_name, reason, order_details, created_at
  FROM risk_event WHERE bot_id = '{bot_id}' AND action = 'REJECT'
  ORDER BY created_at DESC LIMIT 10
  ```
- Check: Is the risk manager rejecting too many orders? Which checks are failing?

### Step 6: Circuit Breaker State
- **Redis MCP**: `GET circuit_breaker:{bot_id}`
- Expected: CLOSED (healthy). OPEN or HALF_OPEN indicate recent failures.
- If OPEN: check when it opened and what triggered it

### Step 7: Market Data Stream Health
- **Redis MCP**: For each market the bot subscribes to:
  ```
  XINFO STREAM market_data:{token_id}
  ```
- Check: Is data flowing? When was the last entry? What's the stream length?

### Step 8: Consumer Group Lag
- **Redis MCP**: For each subscribed market:
  ```
  XINFO GROUPS market_data:{token_id}
  ```
- Check: What's the pending count for this bot's consumer group? High pending = bot can't keep up.

### Step 9: Wallet Balance
- **Redis MCP**: `HGETALL wallet_balance:{wallet_id}`
- Check: Does the bot's wallet have sufficient balance? Is available_balance > 0?

### Step 10: Prometheus Metrics (if Grafana MCP available)
- Query key metrics for this bot:
  - Signal generation rate: `rate(polybot_signals_total{bot_id="{bot_id}"}[5m])`
  - Execution latency: `histogram_quantile(0.95, polybot_execution_latency_seconds_bucket{bot_id="{bot_id}"})`
  - Fill rate: `rate(polybot_fills_total{bot_id="{bot_id}"}[5m])`
  - Error rate: `rate(polybot_errors_total{bot_id="{bot_id}"}[5m])`

### Step 11: Recent Logs
- **Docker MCP**: Read orchestrator container logs, filter for `bot_id`
- Look for: ERROR/WARNING level entries, stack traces, timeout messages

## Output Format

```
BOT DIAGNOSTIC REPORT: {bot_id}
═══════════════════════════════
Timestamp: {now}

 1. Bot Status:         [OK] RUNNING
 2. Container Health:   [OK] orchestrator healthy (up 4h 23m)
 3. Recent Signals:     [OK] 18 signals in last hour (normal rate)
 4. Recent Fills:       [WARN] Only 2 fills in last hour (expected ~10)
 5. Risk Rejections:    [ISSUE] 8 rejections — all "wallet_balance_insufficient"
 6. Circuit Breaker:    [OK] CLOSED
 7. Market Data Stream: [OK] market_data:0x123 — 450 entries/min
 8. Consumer Group Lag: [OK] 0 pending messages
 9. Wallet Balance:     [ISSUE] available_balance: $0.50 (near zero)
10. Prometheus Metrics: [SKIP] Grafana MCP not available
11. Recent Logs:        [OK] No errors in last 30 minutes

DIAGNOSIS
═════════
Root cause: Wallet balance depleted. The bot is generating signals but orders
are being rejected by the risk manager (step 5: "wallet_balance_insufficient").

RECOMMENDED ACTIONS
═══════════════════
1. Check wallet balance: /wallet-audit
2. If funds are available in Vault, rebalance to Alpha tier
3. If total funds are low, deposit more USDC to the wallet
```

## Important
- Always present findings factually — let the user decide on actions
- If multiple issues are found, prioritize by severity (ISSUE > WARN > OK)
- Cross-reference findings: e.g., low fills (step 4) + risk rejections (step 5) + low balance (step 9) = consistent diagnosis
