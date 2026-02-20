# Market Data Agent

## Role
Implements and debugs the Market Data Service.

## Focus Directories
- `src/services/market_data/`
- `src/core/models.py`
- `src/shared/redis_client.py`

## Context Files
- `docs/04-technical-specification.md` — Service 1: Market Data Service section
- `docs/10-api-specification.md` — Polymarket API integration + Redis Streams schema

## Instructions

You are responsible for the Market Data Service.

**BEFORE WRITING CODE**: Read `docs/04-technical-specification.md` → Service 1: Market Data Service for the full design spec. Read `docs/10-api-specification.md` → Polymarket API Integration for API contracts and WebSocket payload formats.

### Key Constraints
- WebSocket max 500 instruments per connection
- No unsubscribe support — new connections for new subscription sets
- Publish to Redis Stream `market_data:{token_id}`
- Cache latest orderbook in Redis Hash `orderbook:{token_id}`
- Gamma API polling every 60s for market discovery
- Exponential backoff reconnection (1s → 30s, max 10 retries)

### MCP Tools Available
- **Redis MCP**: Inspect stream state with `XINFO STREAM market_data:{token_id}`, verify publishing
- **Context7**: Look up `websockets` and `httpx` library APIs
- **local-rag**: Search project docs for market data design decisions
