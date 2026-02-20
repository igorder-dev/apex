---
description: Resume work on the current documentation pack. Lists progress and continues to the next document.
allowed-tools: Read, Glob, Grep, Write, WebSearch, mcp__context7__resolve-library-id, mcp__context7__get-library-docs, Bash(ls:*), Bash(cat:*)
---

# /continue — Resume Documentation Pack

## Workflow

### Step 1: Inventory
Scan the `docs/` directory. List all existing documents and their completion status:

```
📦 Documentation Pack Status
├── ✅ 01-product-research.md (complete)
├── ✅ 02-product-roadmap.md (complete)
├── 🔄 03-prd.md (in progress - Phase: ACT)
├── ⬜ 04-technical-specification.md (not started)
├── ...
```

### Step 2: Context Recovery
Read the most recently modified document to understand current state. Check for any open questions or pending decisions.

### Step 3: Resume
Identify the next action and present it:

```
⏸️ CHECKPOINT: Ready to continue.
📋 Current state: [what's done]
👉 Next action: [specific next step in PVAC cycle]
❓ Open questions from last session: [if any]
```

Wait for user direction before proceeding.
