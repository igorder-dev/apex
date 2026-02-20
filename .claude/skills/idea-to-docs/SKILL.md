---
name: idea-to-docs
description: >
  Generates a complete project documentation pack from a product idea.
  Produces 12 structured markdown documents covering product research,
  roadmap, PRD, technical specification, development guidelines, CLAUDE.md,
  agent configs, security, infrastructure, API, testing, and governance.
  Use when the user wants to convert an idea into a full documentation pack,
  or when generating any individual document in the pack.
---

# Idea-to-Documentation Generation

You are generating project documentation. Follow these rules strictly:

## Document Generation Order
Always generate documents in this order. Each document builds on the previous:
1. Product & Market Research → informs all subsequent documents. **Build the Domain Validation Checklist here.**
2. Product Roadmap → defines scope and phasing
3. PRD → detailed requirements from roadmap
4. Technical Specification → architecture from PRD requirements
5. Development Guidelines → standards for implementing the tech spec
6. CLAUDE.md → project config derived from tech spec + guidelines. **Must reference ALL docs.**
7. Agents & Skills → automation configs derived from all above. **Must tell agents which docs to read.**
8. Security Specification → threat model from tech spec
9. Infrastructure Specification → deployment from tech spec + security
10. API Specification → contracts from PRD + tech spec
11. Testing Strategy → quality assurance from PRD + tech spec
12. Project Governance → process from team size + guidelines

## PVAC Cycle per Document
For each document:
1. **PLAN**: Outline what the document will contain, surface assumptions. Can be abbreviated but never skipped.
2. **VERIFY**: Research any claims, validate technology choices with Context7. Minimum 1 source per major claim.
3. **ACT**: Write the full document using the templates in CLAUDE.md. Run self-review pass before presenting.
4. **CHECK**: Run Mandatory Self-Audit Checklist + Proactive Gap Analysis. Non-negotiable.

## PVAC Enforcement
- **NEVER generate more than 2 documents without a checkpoint**
- **NEVER auto-approve** — wait for explicit user message at every checkpoint
- If user says "proceed" or "looks good", shorten checkpoints but still pause
- If user says "generate all remaining", present batch plan first, then abbreviated checkpoints every 2 docs

## Proactive Gap Analysis (After Every Document)
Before presenting each document, check:
- **Domain gaps**: What would a real PM/Architect flag that isn't covered?
- **Cross-doc gaps**: Does this doc create requirements for future docs that aren't obvious?
- **3 AM test**: What happens when each major component fails?
- **Day 2 problem**: What happens after initial setup/launch?
Present gaps in the checkpoint with suggested additions or deferral targets.

## Mandatory Self-Audit (Every Document)
Before presenting, verify:
1. References all entities from PREVIOUS documents (forward consistency)
2. Anticipates needs of SUBSEQUENT documents (backward planning)
3. Developer reading ONLY this doc wouldn't have unanswered questions
4. Domain Validation Checklist items are addressed (if applicable)
5. "Day 2 problem" is addressed

## Minimum Quality Floor
Every document must meet the minimum research, diagrams, and cross-reference counts defined in CLAUDE.md. Late-stage documents (09-12) get the SAME rigor as early documents (01-04).

## Cross-Reference Integrity
- Every technology mentioned in Tech Spec must appear in Infrastructure Spec
- Every user story in PRD must trace to a feature in Roadmap
- Every API endpoint in API Spec must trace to a requirement in PRD
- Security Spec mitigations must be reflected in Tech Spec and Infrastructure Spec
- Testing Strategy must cover all acceptance criteria from PRD

## Downstream Consumer Principle
Documents serve TWO audiences: humans and AI coding agents.
- Doc 06 (CLAUDE.md) MUST have a "Documentation — READ FIRST" section listing ALL docs
- Doc 07 (Agents) MUST tell each agent role which docs to read
- Generate `docs/llms.txt` as a section-level index for AI agents

## Context Management
- Batch 1 (Docs 01-04): Full PVAC, thorough research
- Batch 2 (Docs 05-08): Medium PVAC, verify against Batch 1
- Batch 3 (Docs 09-12): Standard PVAC, build on established patterns
- Before context fills: write `docs/_session-notes.md` with key decisions and state
- After compaction: read README.md, latest doc, and _session-notes.md before continuing

## Pack Completion — Definition of Done
The pack is NOT complete until Content Completeness, Agent Usability, Operational Readiness, and Domain-Specific checklists all pass (see CLAUDE.md for full checklist).

## File Output
- Write files to `docs/` directory
- Use naming convention: `XX-document-name.md`
- Update `docs/README.md` index after each new document
- Generate `docs/llms.txt` after all documents are complete
- Write `docs/_session-notes.md` when context is filling up
- Include `<!-- STATUS: DRAFT | APPROVED -->` comment at top of each file

## Research Integration
- Use web search for market data, competitor info, technology benchmarks
- Use Context7 MCP for library docs when specifying technical choices
- Always cite sources in research documents
- Flag low-confidence claims explicitly
- Meet minimum research counts per document (see CLAUDE.md quality floor)
