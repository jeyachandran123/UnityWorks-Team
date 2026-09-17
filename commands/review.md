---
description: One UnityWorks specialist reviews the current diff and files an evidence report
argument-hint: <role> [scope]   e.g. security-engineer app/auth
allowed-tools: Bash, Read, Glob, Agent
---

Arguments: $ARGUMENTS

Roles: `security-engineer`, `architecture-auditor`, `vision-specialist`, `backend-engineer`,
`frontend-engineer`, `ux-architect`, `devops-engineer`, `qa-engineer`, `product-analyst`,
`release-scribe`, `ai-ml-architect`. (`tech-lead` triages reports — use `/harden` for that.)

1. The first word of the arguments is the role. If it is missing or not in the list, reply with the
   list and stop.
2. Locate the repositories **by their contents, never by folder name**, and show the diff. If the
   script exits non-zero, relay its message and stop — do not guess a path.
   - Role `ai-ml-architect`:
     ```bash
     R="$(bash "${CLAUDE_PLUGIN_ROOT}/scripts/find-repo" ai-assistant-backend)" && echo "== $R" &&
     git -C "$R" status --short && git -C "$R" diff --stat "$(git -C "$R" merge-base HEAD origin/main)"
     ```
   - Any other role:
     ```bash
     . "${CLAUDE_PLUGIN_ROOT}/scripts/locate-pair" && for r in "$BACKEND" "$FRONTEND"; do
       echo "== $r"; git -C "$r" status --short; git -C "$r" diff --stat "$(git -C "$r" merge-base HEAD origin/main)"; done
     ```
3. Dispatch **one** `unityworks-team:<role>` agent. Its prompt contains, and only contains:
   - the absolute repository path(s) printed by step 2;
   - the scope: the rest of the arguments if given, otherwise "the uncommitted changes plus the commits
     on this branch not on origin/main", with the file list from step 2;
   - today's date (`date +%F`) for the report directory;
   - "File your report per unityworks-team:evidence-report and return its path, verdict and counts."
4. When it returns, read the report file it names and reply with: the verdict, each blocker and major
   finding as one line with its `file:line`, and the report path. Do not start fixing anything.
