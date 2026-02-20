# PolyBot Platform — Documentation Pack

> Automated trading platform for Polymarket prediction markets. Python 3.11+ async microservices with Docker Compose deployment on a single VPS.

**Generated**: 2026-02-15
**Status**: Complete (Phase 1 MVP specification)
**Team Scale**: Solo developer → small team (2-5)

---

## Quick Start

If you're a coding agent or new contributor, start here:

0. **[15-state-template.md](./15-state-template.md)** → Copy to project root as `STATE.md`. Agent reads this first every session.
1. **[13-claude-code-setup.md](./13-claude-code-setup.md)** → Configure Claude Code on VPS: copy CLAUDE.md, AGENTS.md, skills, MCP servers.
2. **[llms.txt](./llms.txt)** → Section-level index of all docs with line ranges. Read this before diving into any specific document.
3. **[06-claude-md.md](./06-claude-md.md)** → Copy to project root as `CLAUDE.md`. Contains build/test commands, architecture overview, and critical gotchas.
4. **[04-technical-specification.md](./04-technical-specification.md)** → Full architecture, data model, service designs, and ADRs.
5. **[05-development-guidelines.md](./05-development-guidelines.md)** → Repository structure, dev setup, code standards, CI/CD.

---

## Document Index

| # | Document | Description | Key Audience |
|---|----------|-------------|-------------|
| 01 | [Product Research](./01-product-research.md) | Market analysis (TAM/SAM/SOM), competitive landscape, risk assessment, Polymarket ecosystem deep-dive | Product/Business |
| 02 | [Product Roadmap](./02-product-roadmap.md) | 6-phase roadmap from Binary Arb MVP through AI/ML ensemble, milestones, exit criteria, wallet architecture | Product/Business |
| 03 | [PRD](./03-prd.md) | 63 user stories across 9 epics, functional/non-functional requirements, acceptance criteria | Product/Engineering |
| 04 | [Technical Specification](./04-technical-specification.md) | System architecture (Mermaid diagrams), 6 ADRs, data model (ER diagram), 6 service designs, Redis topology, binary arb algorithm | Engineering |
| 05 | [Development Guidelines](./05-development-guidelines.md) | Repository structure, dev environment setup, Git workflow, code standards, CI/CD pipeline, performance budget | Engineering |
| 06 | [CLAUDE.md](./06-claude-md.md) | Ready-to-copy CLAUDE.md for coding agents. Build commands, architecture summary, 10 non-obvious gotchas | Engineering (Agents) |
| 07 | [Agents & Skills](./07-agents-md.md) | AGENTS.md with 5 roles, 4 slash commands, 3 sub-agent configs, MCP server setup | Engineering (Agents) |
| 08 | [Security Specification](./08-security-spec.md) | Threat model (STRIDE), authentication/authorization, data protection, log redaction, incident response, pre-launch checklist | Security/Engineering |
| 09 | [Infrastructure Specification](./09-infrastructure-spec.md) | Docker Compose configs, VPS provisioning script, Caddy reverse proxy, Prometheus/Grafana monitoring, backup/recovery, deployment workflow | DevOps/Engineering |
| 10 | [API Specification](./10-api-specification.md) | Dashboard REST + SSE API (all endpoints), Polymarket API integration patterns, inter-service Redis communication, Pydantic models | Engineering |
| 11 | [Testing Strategy](./11-testing-strategy.md) | Test pyramid (unit/integration/E2E), critical test suites, paper trading validation, coverage requirements, quality gates | QA/Engineering |
| 12 | [Project Governance](./12-project-governance.md) | Decision-making, contribution workflow, release management, team scaling guide, compliance, post-mortem template | All |
| 13 | [Claude Code Setup](./13-claude-code-setup.md) | VPS setup for Claude Code: CLAUDE.md, AGENTS.md, skills, MCP servers, sub-agents | Engineering (Agents) |
| 14 | [MCP & Skills Specification](./14-mcp-skills-specification.md) | 7 MCP servers (5 P0 + 2 P1), 10 slash commands, phase-gated rollout, context budget analysis | Engineering (Agents) |
| 15 | [STATE.md Template](./15-state-template.md) | Template for `STATE.md` project state file. Copy to project root. Auto-maintained by agent on every commit. | Engineering (Agents) |
| — | [llms.txt](./llms.txt) | Section-level index with line ranges for all docs — read this first for fast navigation | Engineering (Agents) |
| — | [mcp-config.json](./mcp-config.json) | MCP server configuration (copy to `.mcp.json` in project root) | Engineering (Agents) |
| — | [skills/docs/SKILL.md](./skills/docs/SKILL.md) | `/docs` slash command skill (copy to `.claude/skills/docs/`) | Engineering (Agents) |
| — | [.claude/commands/](./.claude/commands/) | 10 slash command files (copy to `.claude/commands/`) | Engineering (Agents) |
| — | [.claude/agents/](./.claude/agents/) | 4 sub-agent configs (copy to `.claude/agents/`) | Engineering (Agents) |

---

## Architecture at a Glance

```
VPS (Docker Compose)
├── Caddy (reverse proxy, auto-TLS)
├── Dashboard (FastAPI + React SPA)
├── Market Data Service (WebSocket + Gamma API)
├── Orchestrator (bot lifecycle, health checks)
├── Execution Engine (py-clob-client, rate limiting, batching)
├── Risk Manager (10-step pre-trade checks, circuit breakers)
├── Wallet Manager (3-tier risk model, software ledger)
├── PostgreSQL 16 + TimescaleDB (persistence)
├── Redis 7 (Streams + Pub/Sub + Cache)
├── Prometheus + Grafana (monitoring)
└── Bot Modules (pluggable strategies)
     └── Binary Arbitrage (MVP)
```

**MVP Strategy**: Binary Arbitrage (Full-Set Parity) — scans for YES + NO price violations across binary markets, executes FOK orders for mathematically guaranteed profit.

---

## Key Decisions

| Decision | Choice | Document |
|----------|--------|----------|
| Primary language | Python 3.11+ | [ADR-001](./04-technical-specification.md) |
| Database | PostgreSQL 16 + TimescaleDB | [ADR-002](./04-technical-specification.md) |
| Message broker | Redis 7 Streams | [ADR-003](./04-technical-specification.md) |
| Wallet architecture | 3-tier risk model (Vault/Alpha/Sweep) | [ADR-004](./04-technical-specification.md) |
| Dashboard stack | FastAPI + React SPA | [ADR-005](./04-technical-specification.md) |
| Bot interface | BaseBot ABC (10 methods) | [ADR-006](./04-technical-specification.md) |
| MVP strategy | Binary Arbitrage | [01-product-research.md](./01-product-research.md) |

---

## Reading Order

**For understanding the product**: 01 → 02 → 03
**For building the system**: 13 (setup) → 06 → 04 → 05 → 10 → 11
**For deploying**: 09 → 08
**For contributing**: 07 → 12 → 05
**For configuring Claude Code**: 13 → llms.txt → 06 → 07 → 15 (STATE.md template)
