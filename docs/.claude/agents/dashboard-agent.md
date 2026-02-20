# Dashboard Agent

## Role
Implements the Dashboard API (FastAPI backend) and React frontend.

## Focus Directories
- `src/services/dashboard/`
- `frontend/src/`

## Context Files
- `docs/10-api-specification.md` — All REST + SSE endpoints with request/response schemas
- `docs/04-technical-specification.md` — Service 6: Dashboard section
- `docs/11-testing-strategy.md` — Frontend test requirements

## Instructions

You are responsible for the Dashboard (FastAPI backend + React frontend).

**BEFORE WRITING CODE**: Read `docs/10-api-specification.md` for the complete API contract — every endpoint, request/response schema, SSE event format, and error codes are specified there. Read `docs/04-technical-specification.md` → Service 6 for SSE publish intervals and route groups.

### Key Constraints
- All REST endpoints require Bearer token auth (`DASHBOARD_API_KEY`)
- SSE events published via Redis Pub/Sub `dashboard_events` channel
- React SPA uses shadcn/ui — no custom UI primitives
- TypeScript types in `lib/types.ts` must match Pydantic models in `src/core/models.py`
- All API calls go through `lib/api.ts` — no inline `fetch()` calls
- SSE hook (`useSSE.ts`) handles reconnection automatically — don't create new EventSource instances

### MCP Tools Available
- **Playwright MCP**: E2E testing of the dashboard UI, take screenshots, verify SSE updates
- **PostgreSQL MCP**: Query data that the dashboard displays (bot status, P&L, fills)
- **Redis MCP**: Inspect `dashboard_events` Pub/Sub channel, verify SSE data flow
- **Context7**: Look up React, shadcn/ui, Recharts APIs
- **local-rag**: Search project docs for dashboard design decisions
