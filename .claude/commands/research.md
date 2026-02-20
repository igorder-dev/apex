---
description: Deep-dive research on a specific topic relevant to the project. Produces a structured research brief.
allowed-tools: WebSearch, mcp__context7__resolve-library-id, mcp__context7__get-library-docs, Read, Write
---

# /research — Deep Research Brief

## Input
Research topic: $ARGUMENTS

## Workflow

### Step 1: Scope Definition
Define what exactly needs to be researched and why it matters for the project. Identify 3-5 specific questions to answer.

### Step 2: Research Execution
Conduct multiple web searches to cover:
- Current state of the art
- Industry best practices
- Real-world case studies and post-mortems
- Quantitative data (benchmarks, market data, adoption stats)
- Expert opinions and recent developments

Use Context7 when the topic involves specific libraries or frameworks.

### Step 3: Research Brief

```markdown
# Research Brief: [Topic]
**Date**: [today]
**Relevance**: [which document(s) this informs]

## Key Findings
1. [Finding with source]
2. [Finding with source]
3. [Finding with source]

## Detailed Analysis
[Structured analysis organized by sub-topic]

## Data Points
| Metric | Value | Source | Date |
|--------|-------|--------|------|
| ... | ... | ... | ... |

## Implications for This Project
- [How finding X affects our decision on Y]
- [How finding Z changes our approach to W]

## Confidence Level
- High confidence: [topics well-covered by sources]
- Medium confidence: [topics with limited or conflicting data]
- Low confidence: [topics requiring further investigation]

## Sources
1. [Source with URL]
2. [Source with URL]
```

### Step 4: Checkpoint

```
⏸️ CHECKPOINT: Research complete.
📋 Summary: [key takeaway]
👉 Suggested action: [how to apply findings]
```
