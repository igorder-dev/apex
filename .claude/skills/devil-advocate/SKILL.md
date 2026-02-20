---
name: devil-advocate
description: >
  Performs constructive devil's advocate analysis on decisions, technology
  choices, and architectural approaches. Use when evaluating tradeoffs,
  challenging assumptions, or when a decision needs stress-testing before
  committing to it in documentation.
---

# Devil's Advocate Analysis

When performing devil's advocate analysis, follow this structured approach:

## Trigger Conditions
Activate this skill when:
- A technology choice is being made (database, framework, hosting, etc.)
- An architectural decision is being documented in an ADR
- A build-vs-buy decision arises
- The user is confident in an approach that has non-obvious risks
- Feature prioritization involves significant tradeoffs

## Analysis Structure

### 1. Steel-man the current position
Before attacking, present the BEST case for the current choice. This shows fairness and ensures the critique is against a strong version of the argument.

### 2. Identify the three biggest risks
For each risk:
- **Probability**: How likely is this to happen? (cite evidence)
- **Impact**: What happens if it does?
- **Detectability**: Will we know before it's too late?
- **Mitigation**: Can we reduce the risk while keeping the choice?

### 3. Name the best alternative
Not a strawman — the genuine best alternative approach. Explain when it would be the right choice.

### 4. Identify reversibility
- **Easily reversible**: Low-cost to change later → proceed with less analysis
- **Costly to reverse**: Significant migration → more analysis warranted
- **Irreversible**: Permanent commitment → maximum scrutiny required

### 5. Deliver verdict
Always end with a clear, actionable recommendation. Never leave the user in analysis paralysis.

## Tone
- Respectful but direct
- Evidence-based, not opinion-based
- Acknowledge when the current choice IS the best option
- Never artificially inflate risks to seem thorough
