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
1. Product & Market Research → informs all subsequent documents
2. Product Roadmap → defines scope and phasing
3. PRD → detailed requirements from roadmap
4. Technical Specification → architecture from PRD requirements
5. Development Guidelines → standards for implementing the tech spec
6. CLAUDE.md → project config derived from tech spec + guidelines
7. Agents & Skills → automation configs derived from all above
8. Security Specification → threat model from tech spec
9. Infrastructure Specification → deployment from tech spec + security
10. API Specification → contracts from PRD + tech spec
11. Testing Strategy → quality assurance from PRD + tech spec
12. Project Governance → process from team size + guidelines

## PVAC Cycle per Document
For each document:
1. **PLAN**: Outline what the document will contain, surface assumptions
2. **VERIFY**: Research any claims, validate technology choices with Context7
3. **ACT**: Write the full document using the templates in CLAUDE.md
4. **CHECK**: Self-review against quality checklist

## Cross-Reference Integrity
- Every technology mentioned in Tech Spec must appear in Infrastructure Spec
- Every user story in PRD must trace to a feature in Roadmap
- Every API endpoint in API Spec must trace to a requirement in PRD
- Security Spec mitigations must be reflected in Tech Spec and Infrastructure Spec
- Testing Strategy must cover all acceptance criteria from PRD

## File Output
- Write files to `docs/` directory
- Use naming convention: `XX-document-name.md`
- Update `docs/README.md` index after each new document
- Include `<!-- STATUS: DRAFT | APPROVED -->` comment at top of each file

## Research Integration
- Use web search for market data, competitor info, technology benchmarks
- Use Context7 MCP for library docs when specifying technical choices
- Always cite sources in research documents
- Flag low-confidence claims explicitly
