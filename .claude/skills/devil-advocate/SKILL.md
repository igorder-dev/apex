---
name: devil-advocate
description: >
  Performs constructive devil's advocate analysis on decisions, technology
  choices, and architectural approaches. Use when evaluating tradeoffs,
  challenging assumptions, or when a decision needs stress-testing before
  committing to it in documentation.
---

# Devil's Advocate Analysis (Enhanced)

When performing devil's advocate analysis, follow this structured approach:

## Trigger Conditions
Activate this skill when:
- A technology choice is being made (database, framework, hosting, etc.)
- An architectural decision is being documented in an ADR
- A build-vs-buy decision arises
- The user is confident in an approach that has non-obvious risks
- Feature prioritization involves significant tradeoffs
- **After every document generation** (automatic — scope critique)

## Analysis Structure

### Level 1: Decision Critique

#### 1. Steel-man the current position
Before attacking, present the BEST case for the current choice. This shows fairness and ensures the critique is against a strong version of the argument.

#### 2. Identify the three biggest risks
For each risk:
- **Probability**: How likely is this to happen? (cite evidence)
- **Impact**: What happens if it does?
- **Detectability**: Will we know before it's too late?
- **Mitigation**: Can we reduce the risk while keeping the choice?

#### 3. Name the best alternative
Not a strawman — the genuine best alternative approach. Explain when it would be the right choice.

#### 4. Identify reversibility
- **Easily reversible**: Low-cost to change later → proceed with less analysis
- **Costly to reverse**: Significant migration → more analysis warranted
- **Irreversible**: Permanent commitment → maximum scrutiny required

#### 5. Deliver verdict
Always end with a clear, actionable recommendation. Never leave the user in analysis paralysis.

### Level 2: Scope Critique (Run after every document)
Ask these SPECIFIC questions:
- **"What did we FORGET?"** — actively search for missing requirements
- **"What breaks when this fails?"** — failure mode analysis
- **"What does the user need that they didn't ask for?"** — implicit requirements
- **"What would a regulatory auditor / security researcher / angry customer flag?"**

### Level 3: "First Day on the Job" Test
Imagine a developer reading this doc pack on Day 1:
- Can they set up the project without asking questions?
- Do they know which doc to read for which task?
- Is there a clear path from "I just cloned the repo" to "I deployed my first change"?
- Are gotchas and non-obvious constraints highlighted?

## Minimum Gaps Rule
The devil's advocate MUST identify at least 2 potential gaps or concerns per document.
If you genuinely can't find any, state: "No gaps identified — this document is comprehensive."
This forces active gap-seeking rather than rubber-stamping.

## Tone
- Respectful but direct
- Evidence-based, not opinion-based
- Acknowledge when the current choice IS the best option
- Never artificially inflate risks to seem thorough
- Focus on SCOPE gaps (what's missing?) not just DECISION quality (is this the right choice?)
