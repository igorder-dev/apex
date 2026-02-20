---
description: Resume work on the current documentation pack. Lists progress and continues to the next document.
allowed-tools: Read, Glob, Grep, Write, WebSearch, mcp__context7__resolve-library-id, mcp__context7__get-library-docs, Bash(ls:*), Bash(cat:*)
---

# /continue — Resume Documentation Pack

## Workflow

### Step 1: Context Recovery
Read these files in order to recover state:
1. `docs/README.md` — pack index and navigation
2. Most recently modified document in `docs/`
3. `docs/_session-notes.md` — if it exists, contains key decisions and open questions
4. Check the todo list for pending items

### Step 2: Inventory
Scan the `docs/` directory. List all existing documents and their completion status:

```
📦 Documentation Pack Status
├── ✅ 01-product-research.md (complete)
├── ✅ 02-product-roadmap.md (complete)
├── 🔄 03-prd.md (in progress - Phase: ACT)
├── ⬜ 04-technical-specification.md (not started)
├── ...
```

### Step 3: Orientation
Announce your understanding of the current state:
- What decisions have been made
- What's the Domain Validation Checklist (from doc 01)
- Any open questions from previous sessions

### Step 4: Resume
Identify the next action and present it:

```
⏸️ CHECKPOINT: Ready to continue.
📋 Current state: [what's done, key decisions recalled]
👉 Next action: [specific next step in PVAC cycle]
❓ Open questions from last session: [if any]
```

**Wait for user direction before proceeding.**
