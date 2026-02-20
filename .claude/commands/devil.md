---
description: Run a deep devil's advocate analysis on a specific decision, technology choice, or approach. Stress-tests assumptions and identifies scope gaps.
allowed-tools: WebSearch, Read, Glob, Grep, mcp__context7__resolve-library-id, mcp__context7__get-library-docs
---

# /devil — Devil's Advocate Deep Analysis

## Input
Decision or topic to challenge: $ARGUMENTS

## Analysis Framework

### Step 1: State the Current Position
Clearly articulate what is being proposed or decided.

### Step 2: Level 1 — Decision Critique (Three Lenses)

**🎯 Product Lens**
- Is the user need validated or assumed?
- What's the opportunity cost of this choice?
- How does this compare to what competitors chose?
- What if user behavior differs from our assumptions?

**🏗️ Architecture Lens**
- What breaks first at 10x scale?
- What's the migration cost if this is wrong?
- Are we optimizing for the wrong constraint?
- What's the vendor lock-in risk?
- Search for real-world post-mortems of this approach

**⚙️ DevOps Lens**
- What's the operational burden?
- How does this fail at 3 AM?
- What's the recovery time when (not if) it breaks?
- Can the team realistically operate this?

### Step 3: Level 2 — Scope Critique
Ask these SPECIFIC questions:
- **"What did we FORGET?"** — actively search for missing requirements
- **"What breaks when this fails?"** — failure mode analysis
- **"What does the user need that they didn't ask for?"** — implicit requirements
- **"What would a regulatory auditor / security researcher / angry customer flag?"**

### Step 4: Level 3 — "First Day on the Job" Test
If this decision is already documented:
- Can a developer reading the docs set up the project without asking questions?
- Do they know which doc to read for which task?
- Are gotchas and non-obvious constraints highlighted?

### Step 5: Alternative Approaches
Present 2-3 genuine alternatives (not strawmen). For each:
- Core approach
- Key advantages over the current choice
- Key disadvantages
- When this would be the RIGHT choice

### Step 6: Verdict

```
🔴 DEVIL'S ADVOCATE VERDICT
════════════════════════════

Current approach: [approach name]
Risk level: 🟢 Low / 🟡 Medium / 🔴 High

Key risk: [single biggest concern]
Mitigation: [how to reduce the risk if proceeding]

Scope gaps found: [list any missing requirements or concerns]

Best alternative: [if different from current]
When to reconsider: [trigger conditions for revisiting this decision]

Recommendation: [Proceed / Proceed with mitigations / Reconsider / Strongly reconsider]
```

## Minimum Gaps Rule
The analysis MUST identify at least 2 potential gaps or concerns.
If you genuinely can't find any, state: "No gaps identified — this decision is sound."
