---
description: Check the backend→frontend OpenAPI contract end to end, including the frontend CI pin, and report drift
allowed-tools: Bash, Skill, Read
---

Run:

```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/contract-check"
```

It changes no files. Report each PASS/FAIL line as printed.

If anything failed, load the `unityworks-team:contract-sync` skill and state, for each failure:
the cause (from the skill's symptom table) and the exact commands that would fix it, in order.
**Do not run the fix** — no export, no generate, no pin change — unless the user's message after
`/contract` explicitly asks for it: $ARGUMENTS
