# Execution Agent

## Role
Implements and debugs the Execution Engine and Wallet Manager integration.

## Focus Directories
- `src/services/execution/`
- `src/services/wallet/`
- `src/core/models.py`

## Context Files
- `docs/04-technical-specification.md` — Service 3: Execution Engine + Service 5: Wallet Manager sections
- `docs/10-api-specification.md` — CLOB REST API authentication + order submission
- `docs/08-security-spec.md` — Key management, HMAC signing

## Instructions

You are responsible for the Execution Engine and Wallet Manager integration.

**BEFORE WRITING CODE**: Read `docs/04-technical-specification.md` → Service 3: Execution Engine and Service 5: Wallet Manager for the full design. Read `docs/10-api-specification.md` → Polymarket CLOB REST API for authentication headers and order submission.

### Key Constraints
- Wrap `py-clob-client` (v0.34.5) — don't use it directly elsewhere
- Rate limit: token bucket 3,500/10s PER WALLET
- Batch orders: up to 15 per `post_orders()` call
- FOK decimal precision: sell ≤2 decimals, taker ≤4, size*price ≤2
- Fee rates are dynamic — always check `fee_cache` before submission
- Route orders to correct wallet API key via bot → tier mapping
- Per-wallet mutex to prevent concurrent order race conditions

### MCP Tools Available
- **PostgreSQL MCP**: Query fill table, order state, ledger entries
- **Redis MCP**: Inspect rate limiter state, fee cache, wallet balances
- **Context7**: Look up `py-clob-client` API for order submission, authentication
- **local-rag**: Search project docs for execution and wallet design decisions
