---
description: Run a quality review on a specific document in the documentation pack. Checks completeness, consistency, actionability, and gap analysis.
allowed-tools: Read, Glob, Grep, WebSearch
---

# /review — Document Quality Review

## Input
Document to review: $ARGUMENTS

If no argument provided, ask the user which document to review.

## Quality Checklist

Review the document against these criteria and score each 1-5:

### Completeness (weight: 25%)
- [ ] All template sections present and filled
- [ ] No placeholder text remaining
- [ ] All referenced documents/sections exist
- [ ] N/A sections have rationale

### Consistency (weight: 20%)
- [ ] Terminology matches across all pack documents
- [ ] Technology choices align with tech spec
- [ ] User personas match between PRD and research
- [ ] Priority levels are consistent

### Actionability (weight: 20%)
- [ ] A developer could start work without asking questions
- [ ] Success criteria are measurable
- [ ] Acceptance criteria follow Given/When/Then
- [ ] Setup instructions are step-by-step

### Specificity (weight: 10%)
- [ ] No vague language ("best practices", "as needed", "appropriate")
- [ ] Concrete numbers for all targets (performance, coverage, SLAs)
- [ ] Named tools/services instead of generic categories

### Tradeoff Transparency (weight: 10%)
- [ ] Every major decision has alternatives documented
- [ ] Rationale is evidence-based, not opinion-based
- [ ] Risks are explicitly stated

### Proactive Gap Analysis (weight: 15%)
- [ ] Domain Validation Checklist items addressed (or explicitly deferred)
- [ ] "What breaks at 3 AM?" answered for critical components
- [ ] Day 2/Day 30/Day 365 operational concerns addressed
- [ ] Cross-document forward/backward references complete
- [ ] At least 2 potential gaps identified and addressed (or flagged)

## Minimum Quality Floor Check
Verify the document meets the minimum research, diagrams, and cross-reference counts from CLAUDE.md:

| Document | Min Research | Min Diagrams | Min Cross-Refs |
|----------|-------------|-------------|----------------|
| 01 | 3+ web searches | 1 | N/A |
| 02 | Ref doc 01 | 1 | 3+ |
| 03 | Ref docs 01-02 | 1 | 5+ |
| 04 | 2+ Context7 | 3+ | 5+ |
| 05 | Ref doc 04 | 1 | 3+ |
| 06 | Ref docs 04-05 | 0 | Links to ALL |
| 07 | Ref docs 04-06 | 0 | Links to ALL |
| 08 | 1+ web search | 1 | 3+ |
| 09 | Ref docs 04, 08 | 1 | 3+ |
| 10 | Ref docs 03-04 | 1 | 3+ |
| 11 | Ref docs 03-04 | 1 | 3+ |
| 12 | Ref doc 05 | 1 | 2+ |

## Output Format

```
📊 Quality Review: [document name]
Overall Score: [X/5] ([percentage]%)

✅ Strengths:
- [what's working well]

⚠️ Issues Found:
1. [issue] — [suggested fix]
2. [issue] — [suggested fix]

🔍 Gap Analysis:
1. [gap identified] — [severity: critical/medium/low]
2. [gap identified] — [severity]

📏 Quality Floor: [PASS/FAIL] — [details if fail]

🔴 Critical Gaps:
- [anything that blocks development]

Recommendation: [Approve / Revise with minor changes / Requires significant revision]
```
