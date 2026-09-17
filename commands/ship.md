---
description: Go/no-go for releasing this project — gates, declared checks, source state, and the project's release checklist
allowed-tools: Bash, Read, Grep, Glob, Skill, Write
---

Notes from the user: $ARGUMENTS

Change nothing: no fixes, no version bumps, no commits.

## 1. Deterministic first

```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/standup"
bash "${CLAUDE_PLUGIN_ROOT}/scripts/team-context"
bash "${CLAUDE_PLUGIN_ROOT}/scripts/run-check"
```

## 2. Gates

Run the command (third part) of every distinct `gate` line in `.claude/team.conf` of this repository
and of each related repository, from that repository's root. Run independent repositories in parallel
as background commands. With no team.conf, use the lint/test/build commands the project's CI or
`CLAUDE.md` declares, and say so.

## 3. Release checklist

If `.claude/team.conf` names `release-skill = <name>`, load that project skill and execute every
section of it while the gates run. Otherwise apply the generic minimum: working trees clean, branches
pushed, one migration head if migrations exist, production configuration refuses unsafe defaults,
documentation not contradicting the code.

## Rules

- Every row gets PASS, FAIL or **NOT RUN (reason)**. NOT RUN on a blocker row means no-go.
- CI status is **unverified** unless shown.
- Never read `.env` files, never touch a real database or deploy target.

## Verdict

Write it with `unityworks-team:evidence-report` as `<reviews>/<YYYY-MM-DD>/release.md` in this
repository, then reply with **GO / GO AFTER BLOCKERS / NO-GO**, the table of rows, each blocker in one
line with its fix command, and the report path.
