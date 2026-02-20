---
description: Structured comparison of technology options, approaches, or solutions. Uses web search and Context7 for current data.
allowed-tools: WebSearch, mcp__context7__resolve-library-id, mcp__context7__get-library-docs, Read, Write
---

# /compare — Structured Comparison Analysis

## Input
Options to compare: $ARGUMENTS

## Workflow

### Step 1: Identify Options
Parse the user's input. If only a category is given (e.g., "database options"), research and identify the top 3-5 realistic contenders.

### Step 2: Define Evaluation Criteria
Propose criteria relevant to the comparison. Standard criteria:
- **Maturity & community**: GitHub stars, release frequency, community size, Stack Overflow activity
- **Performance**: Benchmarks, latency, throughput (search for current data)
- **Developer experience**: Documentation quality, learning curve, tooling
- **Operational complexity**: Hosting, monitoring, maintenance burden
- **Cost**: Licensing, infrastructure, team training
- **Scalability**: Horizontal scaling, limits, proven scale
- **Security**: CVE history, security features, compliance certifications
- **Team fit**: Current team skills, hiring market, ecosystem

Ask user to approve or modify criteria before proceeding.

### Step 3: Research
Use web search and Context7 to gather current, factual data for each option against each criterion.

### Step 4: Comparison Matrix

```markdown
## Comparison: [Category]

| Criterion | Weight | [Option A] | [Option B] | [Option C] |
|-----------|--------|------------|------------|------------|
| Maturity | [1-5] | [score + evidence] | ... | ... |
| Performance | [1-5] | ... | ... | ... |
| DX | [1-5] | ... | ... | ... |
| Ops Complexity | [1-5] | ... | ... | ... |
| Cost | [1-5] | ... | ... | ... |
| Scalability | [1-5] | ... | ... | ... |
| Security | [1-5] | ... | ... | ... |
| Team Fit | [1-5] | ... | ... | ... |
| **Weighted Total** | | **[score]** | **[score]** | **[score]** |
```

### Step 5: Recommendation

```
📊 COMPARISON VERDICT
═════════════════════

🏆 Recommended: [Option] (score: X/5)

Why: [2-3 sentences with concrete reasoning]

⚠️ Caveat: [when this recommendation changes]

Runner-up: [Option] — better if [specific condition]
```
