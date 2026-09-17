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
2. **If the role is `ai-ml-architect`**, the repository is the AI Assistant backend — the current
   directory if it contains `app/llm/`, otherwise `Unityworks_AI_Assistant/backend` beside or above it.
   Run `git status --short` and `git diff --stat "$(git merge-base HEAD origin/main)"` there, and use
   that repository alone in step 3. **Otherwise** locate the Vision pair and the diff:
   ```bash
   . "${CLAUDE_PLUGIN_ROOT}/scripts/locate-pair" && for r in "$BACKEND" "$FRONTEND"; do
     echo "== $r"; git -C "$r" status --short; git -C "$r" diff --stat "$(git -C "$r" merge-base HEAD origin/main)"; done
   ```
3. Dispatch **one** `unityworks-team:<role>` agent. Its prompt contains, and only contains:
   - the absolute paths of the backend and frontend repositories;
   - the scope: the rest of the arguments if given, otherwise "the uncommitted changes plus the commits
     on this branch not on origin/main", with the file list from step 2;
   - today's date (`date +%F`) for the report directory;
   - "File your report per unityworks-team:evidence-report and return its path, verdict and counts."
4. When it returns, read the report file it names and reply with: the verdict, each blocker and major
   finding as one line with its `file:line`, and the report path. Do not start fixing anything.
