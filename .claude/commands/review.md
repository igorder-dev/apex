---
description: Run a quality review on a specific document in the documentation pack. Checks completeness, consistency, and actionability.
allowed-tools: Read, Glob, Grep, WebSearch
---

# /review — Document Quality Review

## Input
Document to review: $ARGUMENTS

If no argument provided, ask the user which document to review.

## Quality Checklist

Review the document against these criteria and score each 1-5:

### Completeness (weight: 30%)
- [ ] All template sections present and filled
- [ ] No placeholder text remaining
- [ ] All referenced documents/sections exist
- [ ] N/A sections have rationale

### Consistency (weight: 25%)
- [ ] Terminology matches across all pack documents
- [ ] Technology choices align with tech spec
- [ ] User personas match between PRD and research
- [ ] Priority levels are consistent

### Actionability (weight: 20%)
- [ ] A developer could start work without asking questions
- [ ] Success criteria are measurable
- [ ] Acceptance criteria follow Given/When/Then
- [ ] Setup instructions are step-by-step

### Specificity (weight: 15%)
- [ ] No vague language ("best practices", "as needed", "appropriate")
- [ ] Concrete numbers for all targets (performance, coverage, SLAs)
- [ ] Named tools/services instead of generic categories

### Tradeoff Transparency (weight: 10%)
- [ ] Every major decision has alternatives documented
- [ ] Rationale is evidence-based, not opinion-based
- [ ] Risks are explicitly stated

## Output Format

```
📊 Quality Review: [document name]
Overall Score: [X/5] ([percentage]%)

✅ Strengths:
- [what's working well]

⚠️ Issues Found:
1. [issue] — [suggested fix]
2. [issue] — [suggested fix]

🔴 Critical Gaps:
- [anything that blocks development]

Recommendation: [Approve / Revise with minor changes / Requires significant revision]
```
