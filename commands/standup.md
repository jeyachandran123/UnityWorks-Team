---
description: Git state of this repository and its related repositories, review status and team configuration — no agents, no tests
allowed-tools: Bash
---

Run this and nothing else:

```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/standup"
```

Then reply with a short standup, in this order:

1. **Needs action now** — one line each: unpushed or uncommitted work, related repositories not found,
   untriaged reviews, blocker findings, "NOT CONFIGURED". If nothing, say "Nothing blocking."
2. **State** — one line per repository: branch, ahead/behind, uncommitted count.
3. **Suggested next command** — at most one: `/team-init` if not configured, `/check` if checks are
   declared and cross-repository work is uncommitted or unpushed, `/harden` if there is uncommitted
   work and no recent review, `/ship` if everything is clean.

Report only what the script printed. Upstream counts are as of the last fetch; say so if it matters.
