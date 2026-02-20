# Apex: Idea-to-Documentation Agent — Setup Guide

## What is Apex?

Apex is a Claude Code agent that converts raw product ideas into comprehensive, production-ready documentation packs. It combines the expertise of a Top-Tier Product Manager, Solutions Architect, and DevOps Lead into a single agent that iteratively builds 12 structured documents through a Plan → Verify → Act → Check workflow.

## Quick Start

### 1. Prerequisites
- [Claude Code](https://code.claude.com) installed and authenticated
- Node.js ≥ v18 (for Context7 MCP)
- A project directory for your documentation

### 2. Install the Agent

```bash
# Create a new project directory (or use existing)
mkdir my-project && cd my-project

# Copy all agent files into your project
# Option A: Clone from repo (if published)
# Option B: Copy manually:
cp -r path/to/apex/CLAUDE.md ./
cp -r path/to/apex/.claude ./
```

### 3. Configure MCP Servers

```bash
# Add Context7 MCP (required — provides library documentation)
claude mcp add context7 -- npx -y @upstash/context7-mcp

# Optional: Add with API key for higher rate limits
# Get free key at: https://context7.com/dashboard
claude mcp add context7 -- npx -y @upstash/context7-mcp --api-key YOUR_KEY
```

### 4. Verify Setup

```bash
# Start Claude Code
claude

# Check MCP is connected
> /mcp

# Check skills are loaded
> /skills

# Verify commands are available — type / and you should see:
#   /idea, /continue, /review, /status, /devil, /compare, /research
```

### 5. Start Your First Project

```bash
claude
> /idea A SaaS platform that helps small restaurants manage their online ordering, delivery tracking, and customer loyalty programs
```

## File Structure

```
your-project/
├── CLAUDE.md                              # Core agent prompt (READ THIS)
├── .claude/
│   ├── commands/                          # Slash commands
│   │   ├── idea.md                        # /idea — Start new project
│   │   ├── continue.md                    # /continue — Resume work
│   │   ├── review.md                      # /review — Quality review
│   │   ├── status.md                      # /status — Progress dashboard
│   │   ├── devil.md                       # /devil — Devil's advocate
│   │   ├── compare.md                     # /compare — Tech comparison
│   │   └── research.md                    # /research — Deep research
│   ├── skills/                            # Auto-invoked skills
│   │   ├── idea-to-docs/SKILL.md          # Documentation generation
│   │   ├── devil-advocate/SKILL.md        # Tradeoff analysis
│   │   └── tech-research/SKILL.md         # Technology evaluation
│   └── agents/                            # Sub-agents
│       └── research-agent.md              # Parallel research agent
└── docs/                                  # Generated output (created by agent)
    ├── README.md                          # Auto-generated index
    ├── 01-product-research.md
    ├── 02-product-roadmap.md
    ├── ... (up to 12 documents)
    └── 12-project-governance.md
```

## How to Use

### Starting a New Project
```
/idea [describe your idea in 1-3 sentences]
```
The agent will paraphrase your idea, assess viability, ask clarifying questions, and begin the documentation process.

### The PVAC Cycle
Every document goes through four phases with checkpoints:

1. **PLAN** — Agent outlines what the document will contain
   → You review and approve the plan
2. **VERIFY** — Agent researches to validate claims and choices
   → You confirm the research aligns with your vision
3. **ACT** — Agent writes the full document
   → You review the draft
4. **CHECK** — Agent self-audits against quality criteria
   → You give final approval

**You are the final approver at every checkpoint.** The agent will never proceed without your explicit go-ahead.

### Checking Progress
```
/status
```
Shows a dashboard of all 12 documents and their completion status.

### Challenging a Decision
```
/devil [describe the decision to challenge]
```
Triggers a structured devil's advocate analysis from Product, Architecture, and DevOps perspectives.

### Comparing Options
```
/compare React vs Vue vs Svelte for our frontend
```
Produces a weighted comparison matrix with research-backed scores.

### Deep Research
```
/research payment processing options for European restaurants
```
Conducts multi-source research and produces a structured brief.

### Quality Review
```
/review 03-prd.md
```
Runs the full quality checklist against a specific document.

## Customization

### Adjusting for Your Team
The agent adapts its output based on team size. When starting a project, tell the agent:
- "This is a solo project" → Simplified governance, managed services focus
- "Team of 5 developers" → Lightweight process, balanced docs
- "Team of 30 across 4 squads" → Full governance, detailed architecture

### Adding Domain-Specific Knowledge
Create additional skills in `.claude/skills/` for domain expertise:
```
.claude/skills/your-domain/SKILL.md
```

### Modifying Document Templates
Edit the templates directly in `CLAUDE.md` under the "Document Templates" section. The agent follows these templates exactly.

### Adding More Slash Commands
Create new `.md` files in `.claude/commands/`:
```bash
# Example: Add a /sprint command
echo "Plan the next sprint based on the roadmap..." > .claude/commands/sprint.md
```

## Tips for Best Results

1. **Be specific about constraints upfront**: Budget, timeline, team skills, compliance needs
2. **Don't skip checkpoints**: The PVAC cycle exists to catch problems early
3. **Use /devil liberally**: Challenge your own assumptions, not just the agent's
4. **Start with /research for unfamiliar domains**: Build knowledge before documenting
5. **Review documents in order**: Each builds on the previous — gaps compound
6. **Keep context fresh**: Use `/compact` if the conversation gets long, then `/continue`

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Context7 not responding | Check `claude mcp list` — reinstall if missing |
| Agent skips checkpoints | Remind it: "Follow the PVAC cycle. Wait for my approval." |
| Documents lack depth | Provide more context about your domain and constraints |
| Research seems outdated | Ask the agent to search for more recent sources |
| Agent won't challenge you | Say: "Play devil's advocate on this. Be harsh." |

## Output: What You Get

A complete documentation pack of 12 interconnected Markdown files that provide:

- **For stakeholders**: Market research, roadmap, and product requirements
- **For architects**: Technical specification with ADRs and system diagrams
- **For developers**: Guidelines, CLAUDE.md, and testing strategy
- **For ops/security**: Infrastructure spec, security spec, and governance
- **For AI agents**: CLAUDE.md, AGENTS.md, skills, and MCP configurations

All files are cross-referenced, use consistent terminology, and are ready to commit to your repository.
