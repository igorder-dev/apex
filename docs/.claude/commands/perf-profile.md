# /perf-profile — Async Service Performance Profiler

When the user runs `/perf-profile $ARGUMENTS`, profile the specified service's performance.

Extract `{service_name}` from arguments. Valid services: `market_data`, `orchestrator`, `execution`, `risk`, `wallet`, `dashboard`, `all`.

## Prerequisites

- **Grafana MCP** (P1) — for Prometheus metrics queries. If unavailable, falls back to Docker-based checks.
- **Docker MCP** (P0) — for container resource usage
- **PostgreSQL MCP** (P0) — for slow query analysis

## Steps

1. **Read performance budget**: Read `docs/05-development-guidelines.md` → Performance Budget section for target values:
   - API latency: < 200ms p95
   - Trading loop latency: < 50ms p95 (market data → signal → order)
   - WebSocket processing: < 10ms per message
   - Database queries: < 50ms p95
   - Redis commands: < 5ms p95

2. **Query Prometheus metrics** (if Grafana MCP available):

   For the specified service, query these metrics:
   ```promql
   # Request/processing latency
   histogram_quantile(0.50, polybot_{service}_request_duration_seconds_bucket)
   histogram_quantile(0.95, polybot_{service}_request_duration_seconds_bucket)
   histogram_quantile(0.99, polybot_{service}_request_duration_seconds_bucket)

   # Active async tasks (concurrency)
   polybot_{service}_active_tasks

   # Error rate
   rate(polybot_{service}_errors_total[5m])

   # Redis command latency
   histogram_quantile(0.95, polybot_redis_command_duration_seconds_bucket{service="{service}"})

   # DB query latency
   histogram_quantile(0.95, polybot_db_query_duration_seconds_bucket{service="{service}"})
   ```

3. **Compare against budget**: Flag any metric exceeding its target.

4. **Container resource check** (Docker MCP):
   - CPU usage percentage for the service container
   - Memory usage vs limit
   - Restart count (non-zero = instability indicator)

5. **Database analysis** (PostgreSQL MCP, if applicable):
   ```sql
   -- Check for slow queries in the last hour
   SELECT query, calls, mean_exec_time, max_exec_time
   FROM pg_stat_statements
   WHERE mean_exec_time > 50
   ORDER BY mean_exec_time DESC
   LIMIT 10
   ```

6. **Produce performance report**:

```
PERFORMANCE PROFILE: {service_name}
════════════════════════════════════
Timestamp: {now}
Time range: last 5 minutes

LATENCY
  [OK] p50: 12ms (target: —)
  [OK] p95: 38ms (target: <50ms)
  [WARN] p99: 95ms (spiky — investigate)

THROUGHPUT
  [OK] Requests/sec: 45
  [OK] Active tasks: 3 (no task starvation)

ERROR RATE
  [OK] 0.02% (2 errors in last 5 min)

DEPENDENCIES
  [OK] Redis p95: 2ms (target: <5ms)
  [WARN] DB p95: 62ms (target: <50ms) — check slow queries below

CONTAINER RESOURCES
  [OK] CPU: 15% of 2 cores
  [OK] Memory: 180MB / 512MB limit (35%)
  [OK] Restarts: 0

SLOW QUERIES (if any)
  1. SELECT * FROM fill WHERE bot_id = ... — avg 68ms, max 120ms
     Suggestion: Add index on (bot_id, created_at)

BOTTLENECK SUMMARY
  Primary bottleneck: Database queries exceeding target
  Suggested action: Review indexes on fill and signal tables
```

## Notes
- If Grafana MCP is not installed, the command still provides value via Docker resource checks and PostgreSQL slow query analysis
- For `all` services, run each service sequentially and present a comparison table at the end
- Trading-critical services (execution, risk) should have stricter latency targets than support services (dashboard)
