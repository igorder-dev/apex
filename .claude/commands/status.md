---
description: Show the current progress of the documentation pack with completion status for each document.
allowed-tools: Read, Glob, Bash(ls:*), Bash(wc:*)
---

# /status — Documentation Pack Progress

Scan the `docs/` directory and display a progress dashboard:

```
📦 [Project Name] — Documentation Pack Status
═══════════════════════════════════════════════

Progress: [X/12] documents complete ([percentage]%)

 # | Document                  | Status | Words | Last Modified
---|---------------------------|--------|-------|---------------
 01 | Product Research          | ✅/🔄/⬜ | ... | ...
 02 | Product Roadmap           | ✅/🔄/⬜ | ... | ...
 03 | PRD                       | ✅/🔄/⬜ | ... | ...
 04 | Technical Specification   | ✅/🔄/⬜ | ... | ...
 05 | Development Guidelines    | ✅/🔄/⬜ | ... | ...
 06 | CLAUDE.md                 | ✅/🔄/⬜ | ... | ...
 07 | Agents & Skills           | ✅/🔄/⬜ | ... | ...
 08 | Security Specification    | ✅/🔄/⬜ | ... | ...
 09 | Infrastructure Spec       | ✅/🔄/⬜ | ... | ...
 10 | API Specification         | ✅/🔄/⬜ | ... | ...
 11 | Testing Strategy          | ✅/🔄/⬜ | ... | ...
 12 | Project Governance        | ✅/🔄/⬜ | ... | ...

⬜ = Not started | 🔄 = In progress | ✅ = Complete

❓ Open Questions: [count]
📝 Pending Decisions: [count]
👉 Suggested Next: [next document to work on]
```

Status determination:
- ✅ Complete: File exists AND has been through CHECK phase (look for "APPROVED" marker)
- 🔄 In Progress: File exists but incomplete or not yet approved
- ⬜ Not Started: File does not exist
