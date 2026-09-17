---
description: Run this project's declared checks (e.g. /check contract) and report PASS/FAIL — changes no files
argument-hint: [check name]   (omit to run all)
allowed-tools: Bash, Skill, Read
---

Run:

```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/run-check" $ARGUMENTS
```

Report each check's PASS/FAIL and the failing lines exactly as printed.

- "No checks declared" → say so, and that `check = <name> :: <command>` lines in `.claude/team.conf`
  add them (`/team-init` can propose some).
- A failure → if a project skill in `.claude/skills/` covers that area (its description mentions the
  check or what it verifies), load it and state the cause and the exact fix commands, in order.
  **Do not run the fix** unless the user asks.
