---
description: One specialist reviews the current changes and files an evidence report
argument-hint: <role> [scope]   e.g. security-engineer src/auth
allowed-tools: Bash, Read, Glob, Agent
---

Arguments: $ARGUMENTS

1. **Project context:**
   ```bash
   bash "${CLAUDE_PLUGIN_ROOT}/scripts/team-context"
   ```
   It prints `REPO_SELF`, `TEAM_REVIEWS`, `REPO_<LABEL>` for related repositories, and `TEAM_MISSING`.
   If it fails, relay its message and stop.

2. **Role.** The first word of the arguments. Available:
   - plugin roles: `security-engineer`, `architecture-auditor`, `backend-engineer`,
     `frontend-engineer`, `ux-architect`, `devops-engineer`, `qa-engineer`, `product-analyst`,
     `release-scribe`, `ai-ml-architect` → agent type `unityworks-team:<role>`
   - project roles: each `agent = <name>` in `$REPO_SELF/.claude/team.conf` → agent type `<name>`

   If the role is missing or unknown, reply with that list and stop.

3. **Diff** of each repository (this one, plus related ones if the role is cross-repository —
   devops-engineer, security-engineer, product-analyst, release-scribe, architecture-auditor):
   ```bash
   r=<repo path>; git -C "$r" status --short
   base="$(git -C "$r" merge-base HEAD origin/HEAD 2>/dev/null || git -C "$r" merge-base HEAD origin/main 2>/dev/null)"
   [ -n "$base" ] && git -C "$r" diff --stat "$base"
   ```

4. **Dispatch one agent.** Its prompt contains, and only contains:
   - the absolute repository path(s) from step 1 and, for each, its reviews directory;
   - the scope: the rest of the arguments if given, otherwise "the uncommitted changes plus the commits
     not on the default branch", with the file list from step 3;
   - today's date (`date +%F`);
   - "File your report per unityworks-team:evidence-report and return its path, verdict and counts."

5. When it returns, read the report file and reply with: the verdict, each blocker and major finding as
   one line with its `file:line`, and the report path. Do not start fixing anything.
