---
description: Git state of both Vision AI repos, contract pin health, and unaddressed review findings — no agents, no tests
allowed-tools: Bash
---

Run this and nothing else:

```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/standup"
```

Then reply with a short standup, in this order:

1. **Needs action now** — each item one line: unpushed or uncommitted work, a stale or unpushed schema
   pin, untriaged reviews, blocker findings. If nothing, say "Nothing blocking."
2. **State** — one line per repo: branch, ahead/behind, uncommitted count.
3. **Suggested next command** — at most one: `/contract` if the schema moved, `/harden` if there is
   uncommitted security-surface work and no recent review, `/ship` if everything is clean.

Report only what the script printed. Upstream counts are as of the last fetch; say so if it matters.
