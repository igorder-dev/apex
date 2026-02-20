---
name: tech-research
description: >
  Conducts structured technology research and comparisons using web search
  and Context7 MCP. Use when evaluating technology options for the tech stack,
  comparing frameworks or libraries, or when the technical specification
  needs evidence-based technology choices.
---

# Technology Research

When researching technology options, follow this approach:

## Research Methodology

### Step 1: Define evaluation criteria BEFORE searching
Do not search randomly. Establish what matters for THIS project:
- Performance requirements from the PRD/NFRs
- Team skills and hiring market
- Budget constraints
- Scale requirements
- Compliance needs

### Step 2: Systematic search pattern
For each technology option:
1. **Official documentation** (via Context7 when available)
2. **GitHub metrics**: Stars, issues, PR velocity, last release date
3. **Benchmarks**: Performance comparisons (prefer independent benchmarks)
4. **Production usage**: Who uses this at scale? Any public post-mortems?
5. **Community health**: Stack Overflow activity, Discord/forum engagement

### Step 3: Structured comparison
Always produce a comparison table with weighted scoring.
Weight criteria based on project priorities, not generic importance.

### Step 4: Total Cost of Ownership
Go beyond licensing:
- Learning curve (team ramp-up time)
- Operational cost (hosting, monitoring, maintenance)
- Migration cost (if switching later)
- Hiring cost (availability of experienced developers)

## Context7 Usage
- Always resolve library ID first
- Query with specific use cases, not generic terms
- Use to verify: API availability, configuration options, integration patterns
- Cross-reference Context7 docs with web search for completeness

## Output Standards
- Every claim must have a source
- Distinguish between facts and opinions
- Flag outdated information (>12 months old)
- Note when data is sparse or conflicting
