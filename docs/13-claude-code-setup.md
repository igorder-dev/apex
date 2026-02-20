# Claude Code Setup Guide: PolyBot Platform

## Overview

This guide configures Claude Code on the VPS for developing and operating the PolyBot platform. After setup, coding agents will have:

- **`CLAUDE.md`** — Project context auto-loaded on every session (build commands, architecture, gotchas)
- **`AGENTS.md`** — Agent role definitions for multi-agent workflows
- **10 slash commands** — `/new-bot`, `/check-risk`, `/test-strategy`, `/deploy`, `/debug-bot`, `/e2e-test`, `/wallet-audit`, `/market-scan`, `/perf-profile`, `/backtest`
- **`/docs` skill** — Slash command for targeted documentation lookup
- **5 MCP servers** (P0) — `local-rag`, `context7`, `postgres`, `redis`, `docker`
- **4 sub-agents** — market-data, execution, risk, dashboard

Full MCP & skills specification: `docs/14-mcp-skills-specification.md`

---

## Prerequisites

| Requirement | Version | Check Command |
|-------------|---------|--------------|
| Node.js | 18+ | `node --version` |
| npm | 9+ | `npm --version` |
| Git | 2.30+ | `git --version` |
| Claude Code CLI | Latest | `claude --version` |

---

## Step 1: Install Node.js (if not present)

Node.js is required for MCP servers (`mcp-local-rag` and `context7`) which run via `npx`.

```bash
# Ubuntu/Debian (VPS)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify
node --version   # Should be 18+
npm --version    # Should be 9+
```

---

## Step 2: Install Claude Code CLI

```bash
# Install via npm (recommended for VPS)
npm install -g @anthropic-ai/claude-code

# Verify
claude --version

# Authenticate (first time only)
claude auth login
```

---

## Step 3: Clone the Repository

```bash
git clone <repository-url> polybot
cd polybot
```

---

## Step 4: Copy Project Configuration Files

The `docs/` folder contains all the files that need to be placed in the project root and `.claude/` directory.

```bash
# Project root files
cp docs/06-claude-md.md CLAUDE.md
# Extract AGENTS.md content from docs/07-agents-md.md (between the ```markdown fences)
# Or create AGENTS.md manually — see docs/07-agents-md.md for the content

# MCP configuration (project-scoped)
cp docs/mcp-config.json .mcp.json

# Claude Code skills
mkdir -p .claude/skills/docs
cp docs/skills/docs/SKILL.md .claude/skills/docs/SKILL.md

# Slash commands (10 commands)
cp -r docs/.claude/commands/ .claude/commands/

# Sub-agents (4 agents)
cp -r docs/.claude/agents/ .claude/agents/

# Project state tracking
cp docs/15-state-template.md STATE.md
# Edit STATE.md: set current phase from roadmap, first feature from PRD
```

### File Placement Summary

| Source (in docs/) | Destination | Purpose |
|-------------------|-------------|---------|
| `docs/06-claude-md.md` | `./CLAUDE.md` | Auto-loaded project context |
| `docs/07-agents-md.md` | `./AGENTS.md` | Agent role definitions |
| `docs/mcp-config.json` | `./.mcp.json` | MCP server configuration |
| `docs/skills/docs/SKILL.md` | `.claude/skills/docs/SKILL.md` | `/docs` slash command |
| `docs/.claude/commands/*.md` | `.claude/commands/*.md` | 10 slash commands |
| `docs/.claude/agents/*.md` | `.claude/agents/*.md` | 4 sub-agent configs |
| `docs/15-state-template.md` | `./STATE.md` | Session continuity / project state tracking |

---

## Step 5: Initialize MCP Servers

The `.mcp.json` file configures 5 MCP servers (P0). On first run, each will download its dependencies.

```bash
# Trigger first-time model download for mcp-local-rag (~90MB embedding model)
# This runs the server briefly to cache the model
npx -y mcp-local-rag &
MCP_PID=$!
sleep 30
kill $MCP_PID 2>/dev/null

# Verify MCP servers are configured
claude mcp list
```

Expected output should show:
- `local-rag` — Local RAG server pointing to `./docs`
- `context7` — Context7 library documentation server
- `postgres` — PostgreSQL database inspection (restricted/read-only mode)
- `redis` — Redis stream, cache, and pub/sub inspection
- `docker` — Docker container management and log inspection

### Alternative: Add MCP Servers via CLI

If you prefer to configure MCP servers via the CLI instead of `.mcp.json`:

```bash
# Documentation servers
claude mcp add local-rag --scope project \
  --env BASE_DIR=./docs \
  -- npx -y mcp-local-rag

claude mcp add context7 --scope project \
  -- npx -y @upstash/context7-mcp

# Development & debugging servers (P0)
claude mcp add postgres --scope project \
  -- npx -y @crystaldba/postgres-mcp
# Set DATABASE_URL env: postgresql://polybot:password@localhost:5432/polybot
# Set PG_MCP_MODE env: restricted (read-only for safety)

claude mcp add redis --scope project \
  -- npx -y @redis/mcp-redis
# Set REDIS_URL env: redis://localhost:6379/0

claude mcp add docker --scope project \
  -- npx -y @quantgeekdev/docker-mcp

# P1 servers — add after MVP is running (Week 7+)
# claude mcp add grafana --scope project -- npx -y @grafana/mcp-grafana
# claude mcp add sequential-thinking --scope project -- npx -y @modelcontextprotocol/server-sequential-thinking
```

### MCP Server Connection Requirements

| Server | Requires | Notes |
|--------|----------|-------|
| `postgres` | Docker Compose running (PostgreSQL container) | Dev compose exposes port 5432 |
| `redis` | Docker Compose running (Redis container) | Dev compose exposes port 6379 |
| `docker` | Docker socket access | Standard on VPS |
| `local-rag` | First-run model download (~90MB) | Fully offline after |
| `context7` | Internet connection | Fetches from Context7 CDN |

---

## Step 6: Verify Setup

### 6.1 Check CLAUDE.md is Loaded

```bash
claude "What build commands are available for this project?"
```

Expected: Claude should reference `make dev`, `make test`, `docker compose up -d`, etc. from the CLAUDE.md file.

### 6.2 Test the /docs Skill

```bash
claude "/docs how does the circuit breaker work?"
```

Expected: Claude reads `docs/llms.txt`, identifies the relevant section (Service 4: Risk Manager in the tech spec), and returns the circuit breaker FSM details.

### 6.3 Test mcp-local-rag Semantic Search

```bash
claude "Use the local-rag tool to search for 'FOK decimal precision requirements'"
```

Expected: Claude uses `query_documents` to find relevant chunks about FOK order decimal precision from the tech spec and CLAUDE.md gotchas.

### 6.4 Test Context7 (External Library Docs)

```bash
claude "Use Context7 to look up the py-clob-client create_order function signature"
```

Expected: Claude resolves the library ID and returns current API documentation for py-clob-client.

### 6.5 Test PostgreSQL MCP (requires Docker Compose running)

```bash
claude "Use the postgres MCP to list all tables in the public schema"
```

Expected: Claude queries the database and returns the table list (wallets, bots, fills, signals, etc.).

### 6.6 Test Redis MCP (requires Docker Compose running)

```bash
claude "Use the redis MCP to check if the emergency_stop key exists"
```

Expected: Claude queries Redis and reports the key's value (or that it doesn't exist).

### 6.7 Test Docker MCP

```bash
claude "Use the docker MCP to list all running containers"
```

Expected: Claude lists all PolyBot containers with their status and health.

---

## Documentation Navigation Hierarchy

When working with PolyBot documentation, agents should follow this lookup order:

```
1. docs/llms.txt        ← Read first (~3K tokens). Section-level index with line ranges.
                           Identifies WHICH doc and WHICH section to read.

2. /docs skill          ← Use for targeted lookups. Reads llms.txt, matches query,
                           loads only the relevant sections via Read with offset/limit.

3. mcp-local-rag        ← Use for semantic/fuzzy queries when exact section isn't clear.
                           query_documents tool searches across all indexed docs.

4. Direct file read     ← Last resort. Read the full document only when you need
                           comprehensive context (e.g., implementing a complete service).
```

### When to Use What

| Situation | Use |
|-----------|-----|
| "Which doc covers X?" | Read `docs/llms.txt` |
| "Show me the risk manager design" | `/docs risk manager` — loads specific section |
| "How does the order state machine transition on timeout?" | `mcp-local-rag` `query_documents` — semantic search finds the exact passage |
| "I need to implement Service 3 from scratch" | Read full `docs/04-technical-specification.md` Service 3 section (L513-574) |
| "What's the py-clob-client API for batch orders?" | Context7 — external library documentation |
| "What's PolyBot's batch order implementation design?" | `/docs batch orders` — project documentation |

---

## MCP Server Details

### mcp-local-rag

| Property | Value |
|----------|-------|
| **Purpose** | Semantic search over project documentation |
| **Scope** | Project-level (`.mcp.json`) |
| **Storage** | LanceDB file-based vector DB (created in project directory) |
| **Embedding Model** | `all-MiniLM-L6-v2` via Transformers.js (~90MB, downloaded once) |
| **Search Method** | Hybrid: semantic vector similarity + keyword boost |
| **External Dependencies** | None. Fully offline after first model download. |
| **Indexed Directory** | `./docs` (all `.md` files) |

**Available Tools:**

| Tool | Description |
|------|-------------|
| `ingest_file` | Index a single file into the vector store |
| `ingest_data` | Index raw text data |
| `query_documents` | Semantic search across indexed documents |
| `list_files` | List all indexed files |
| `delete_file` | Remove a file from the index |
| `status` | Check index health and statistics |

### Context7

| Property | Value |
|----------|-------|
| **Purpose** | External library/framework documentation lookup |
| **Scope** | Project-level (`.mcp.json`) |
| **External Dependencies** | Internet connection (fetches docs from Context7 CDN) |

**Available Tools:**

| Tool | Description |
|------|-------------|
| `resolve-library-id` | Convert library name to Context7 ID (call first) |
| `get-library-docs` | Fetch documentation for a resolved library |

**Common Library Lookups:**

| Library | Use For |
|---------|---------|
| `py-clob-client` | Polymarket CLOB SDK — order creation, signing, API auth |
| `FastAPI` | Dashboard backend — routes, SSE, middleware |
| `SQLAlchemy` | Database ORM — async sessions, models, queries |
| `Redis (redis-py)` | Redis client — Streams, Pub/Sub, caching |
| `Pydantic` | Data validation — model definitions, serialization |
| `React` | Dashboard frontend — components, hooks, state |
| `shadcn/ui` | UI component library — buttons, tables, cards |
| `Recharts` | Chart library — line charts, area charts for P&L |
| `structlog` | Structured logging — JSON output, context binding |
| `pytest` | Test framework — fixtures, parametrize, async tests |
| `Alembic` | Database migrations — revision management |
| `Docker Compose` | Container orchestration — service definitions, networking |

---

## Troubleshooting

### MCP server fails to start

```bash
# Check if npx can run the server
npx -y mcp-local-rag --help

# If Node.js modules are corrupted
rm -rf ~/.npm/_npx
npx -y mcp-local-rag
```

### Embedding model download fails

```bash
# The model is cached in the npm package cache
# Clear and retry
npm cache clean --force
npx -y mcp-local-rag
```

### CLAUDE.md not loading

```bash
# Verify file exists at project root
ls -la CLAUDE.md

# Verify you're in the project directory when running claude
pwd
claude "What project is this?"
```

### /docs skill not found

```bash
# Verify skill file exists
ls -la .claude/skills/docs/SKILL.md

# Skills are discovered automatically — no registration needed
# Try restarting Claude Code
claude /docs "test query"
```

### Context7 returns no results

```bash
# Context7 requires internet access
# Verify connectivity
curl -s https://context7.com > /dev/null && echo "OK" || echo "No connection"

# For offline work, rely on docs/llms.txt and mcp-local-rag instead
```

---

## Quick Reference Card

```
# Start Claude Code in project directory
cd /path/to/polybot
claude

# Documentation lookup
/docs "circuit breaker design"
/docs "wallet architecture"
/docs "API endpoint for bot management"

# Semantic search (when /docs doesn't match well)
"Use local-rag to search for 'order reconciliation after disconnect'"

# External library docs
"Use Context7 to check FastAPI SSE implementation"

# Development commands
/new-bot market_maker          # Scaffold a new bot strategy
/check-risk                    # Audit risk configuration
/test-strategy binary_arbitrage # Run strategy tests with coverage
/deploy                        # Build and deploy (with safety gates)

# Debugging commands
/debug-bot arb_01              # 11-step bot diagnostic
/wallet-audit                  # Financial reconciliation
/perf-profile execution        # Profile service performance

# Testing & analysis
/e2e-test dashboard            # Run Playwright E2E tests
/market-scan                   # Scan for arbitrage opportunities
/backtest binary_arbitrage     # Historical strategy backtest

# Build & test
make dev          # Start all services with hot reload
make test         # Run all tests
make test-unit    # Unit tests only (fast)
```

---

## Cross-References

- **Project context**: `docs/06-claude-md.md` (CLAUDE.md source)
- **Agent roles & skills**: `docs/07-agents-md.md` (AGENTS.md source)
- **MCP & skills specification**: `docs/14-mcp-skills-specification.md` (complete tooling spec)
- **Development setup**: `docs/05-development-guidelines.md` → Development Environment Setup
- **Infrastructure**: `docs/09-infrastructure-spec.md` → VPS Provisioning
- **Documentation index**: `docs/llms.txt` (section-level index for all docs)
