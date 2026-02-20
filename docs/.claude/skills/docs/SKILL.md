---
name: docs
description: Search and retrieve sections from the PolyBot project documentation
---

# /docs — Documentation Lookup

When the user runs `/docs {query}`, find and return the relevant documentation sections.

## Instructions

1. **Read the index**: Read `docs/llms.txt` — this is a lightweight section-level index of all 12 project documents with line ranges and keyword tags.

2. **Match the query**: Identify the 1-3 most relevant sections based on:
   - Section titles and descriptions matching the query keywords
   - Keyword tags in square brackets (e.g., `[circuit breaker, FSM]`)
   - Document purpose descriptions

3. **Load targeted sections**: For each matching section, use the `Read` tool with `offset` and `limit` parameters based on the line ranges in the index. For example, if the index says `Service 4: Risk Manager (L576-644)`, read `docs/04-technical-specification.md` with `offset: 576` and `limit: 68`.

4. **Present results**: Show the retrieved content with:
   - The source document name and section title
   - The actual content
   - Cross-references to related sections (check the "Cross-References" section at the bottom of each doc)

## Query Routing Quick Reference

Use this to speed up common queries:

| Query Keywords | Go To |
|---------------|-------|
| architecture, system design, components | `docs/04-technical-specification.md` → Architecture Overview |
| ADR, decision, technology choice | `docs/04-technical-specification.md` → ADRs |
| data model, database, tables, schema, ER | `docs/04-technical-specification.md` → Data Model |
| market data, WebSocket, Gamma, orderbook | `docs/04-technical-specification.md` → Service 1 |
| orchestrator, bot lifecycle, health check | `docs/04-technical-specification.md` → Service 2 |
| execution, order, CLOB, rate limit, batch | `docs/04-technical-specification.md` → Service 3 |
| risk, circuit breaker, emergency stop, pre-trade | `docs/04-technical-specification.md` → Service 4 |
| wallet, EOA, ledger, rebalance, tier | `docs/04-technical-specification.md` → Service 5 |
| dashboard API, SSE, React | `docs/04-technical-specification.md` → Service 6 |
| binary arb, full-set parity, arbitrage algorithm | `docs/04-technical-specification.md` → Binary Arb Strategy |
| API endpoint, REST, request, response | `docs/10-api-specification.md` |
| Polymarket API, CLOB, authentication, HMAC | `docs/10-api-specification.md` → Polymarket API Integration |
| Redis, streams, pub/sub, cache | `docs/10-api-specification.md` → Inter-Service Communication |
| Pydantic model, type, contract | `docs/10-api-specification.md` → Pydantic Models |
| user story, acceptance criteria, requirement | `docs/03-prd.md` |
| roadmap, phase, milestone, timeline | `docs/02-product-roadmap.md` |
| test, coverage, fixture, pytest | `docs/11-testing-strategy.md` |
| Docker, Compose, VPS, deploy, Caddy | `docs/09-infrastructure-spec.md` |
| security, threat, auth, encryption, STRIDE | `docs/08-security-spec.md` |
| repo structure, git, CI/CD, code standard | `docs/05-development-guidelines.md` |
| governance, PR, release, contribution | `docs/12-project-governance.md` |
| build, test command, make, gotcha | `docs/06-claude-md.md` |
| agent role, slash command, sub-agent, MCP | `docs/07-agents-md.md` |
| market research, competitor, TAM | `docs/01-product-research.md` |

## Fallback

If the query doesn't match any section clearly:
1. Check if `mcp-local-rag` is available — use `query_documents` for semantic search
2. If not, list the top 3 most likely documents and ask the user to narrow down

## Examples

- `/docs "how does the circuit breaker work?"` → Read tech spec Service 4: Risk Manager (L576-644)
- `/docs "wallet architecture"` → Read tech spec Service 5: Wallet Manager (L646-747) + roadmap Wallet Architecture (L21-87)
- `/docs "FOK decimal precision"` → Read CLAUDE.md Gotchas (L118-138) + tech spec Service 3: Execution Engine (L513-574)
- `/docs "how to deploy"` → Read infra spec Deployment Workflow (L1290-1386)
- `/docs "dashboard API endpoints"` → Read API spec Bot Management (L241-555) + System Endpoints (L115-239)
