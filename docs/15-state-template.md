# STATE.md Template

> Copy this file to the project root as `STATE.md`.
> The development agent maintains this file automatically — updated on every commit.

---

```markdown
# STATE.md — PolyBot Project State

> Auto-maintained by the development agent. Updated on every commit.
> Last updated: YYYY-MM-DD HH:MM UTC
> Last commit: <short-hash> <commit message>

## Current Focus

**Phase**: [Phase 1: Foundation + Binary Arb MVP]
**Feature**: [US-XXX — short description]
**Branch**: [feat/US-XXX-description]
**Status**: [implementing | testing | awaiting-demo | awaiting-approval | blocked]

## Completed (Recent)

| Commit | Feature | Date |
|--------|---------|------|
| — | (none yet) | — |

> Keep last 10 entries only. Older history is in `git log`.

## In Progress

- **US-XXX — [description]**
  - [ ] Read docs (tech spec, PRD acceptance criteria)
  - [ ] Implement core logic
  - [ ] Write unit tests
  - [ ] Run make test-unit (pass)
  - [ ] Demo to user
  - [ ] User approval
  - [ ] Commit

## Up Next

1. US-XXX — [description] (P0)
2. US-XXX — [description] (P0)
3. US-XXX — [description] (P1)

> Max 3 items. Source: `docs/03-prd.md` priorities.

## Blockers / Pending Decisions

- [None]

## Session Notes

[One-line context for next session. Example: "WebSocket reconnection logic implemented but not yet tested against real Polymarket feed."]
```

---

## Usage Instructions

### When to Create

On first project setup, copy this template to the project root:

```bash
cp docs/15-state-template.md STATE.md
```

Then populate it:
1. Set **Phase** from `docs/02-product-roadmap.md` (current phase)
2. Set **Feature** from `docs/03-prd.md` (first P0 user story)
3. Fill **Up Next** with the next 3 user stories from the PRD

### How to Maintain

**On every commit**: Update STATE.md before or alongside the commit:
1. Move completed feature to the "Completed" table (include commit hash)
2. Advance "In Progress" to the next item from "Up Next"
3. Refresh the checklist for the new in-progress item
4. Update timestamps and "Last commit" line
5. Write a "Session Notes" line for the next session

**On session start**: Read STATE.md + `git log --oneline -5` to orient before doing any work.

### Token Budget

STATE.md should stay under **500 tokens** when read by the agent. Keep it scannable:
- "Completed" table: max 10 rows (trim oldest)
- "Up Next": max 3 items
- "Session Notes": 1-2 sentences max
- No prose paragraphs — use bullet points and tables
