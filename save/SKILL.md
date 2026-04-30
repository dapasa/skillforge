---
name: save
description: >
  Saves the current session to engram via mem_session_summary.
  Trigger: When the user invokes /save or asks to save the session.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
allowed-tools: mcp__plugin_engram_engram__mem_session_summary
---

## When to Use

- User invokes `/save`
- User says "save the session", "guardá la sesión", "save progress"

## Critical Patterns

- ONLY action allowed: call `mem_session_summary`
- Do NOT commit code
- Do NOT push to remote
- Do NOT write any files
- Do NOT do anything else beyond the single `mem_session_summary` call

## Steps

1. Derive the project name from the current working directory (use the folder name if no explicit project name is available from context).
2. Call `mem_session_summary` with `project` set to that name and a summary structured as:

```
## Goal
[What we were working on this session]

## Instructions
[User preferences or constraints discovered — skip if none]

## Discoveries
- [Technical findings, gotchas, non-obvious learnings]

## Accomplished
- [Completed items with key details]

## Next Steps
- [What remains to be done — for the next session]

## Relevant Files
- path/to/file — [what it does or what changed]
```

3. After the call succeeds, reply to the user with exactly one short confirmation line, no fluff. Example: "Session saved to engram."

## Constraints

| Action | Allowed |
|--------|---------|
| `mem_session_summary` | YES |
| Git commit | NO |
| Git push | NO |
| Write files | NO |
| Any other tool | NO |
