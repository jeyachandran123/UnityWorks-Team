---
description: Parallel specialist sweep of this project, then tech-lead triage into one ranked remediation plan
argument-hint: [area]   an area from the project's harden rosters (default: whole branch)
allowed-tools: Bash, Read, Glob, Agent
---

Area: $ARGUMENTS

## 1. Establish the ground

```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/standup"
bash "${CLAUDE_PLUGIN_ROOT}/scripts/team-context"
date +%F
```

Record every repository path (`REPO_SELF`, `REPO_<LABEL>`), each one's reviews directory, today's date,
and the refs under review (`git rev-parse --short HEAD`, plus "working tree" if uncommitted).

## 2. Choose the roster

Read the `harden = <area> :: <roles…>` lines in `$REPO_SELF/.claude/team.conf`.

- An area was given and a line matches it → that roster.
- No area → the `default` line.
- No matching line (or no team.conf) → pick 3–5 from `security-engineer`, `architecture-auditor`,
  `backend-engineer`, `frontend-engineer`, `ux-architect`, `devops-engineer`, `qa-engineer`,
  `product-analyst`, `release-scribe`, plus any `agent = <name>` project agents, whose descriptions
  match the area or the changed files. Say which and why.

Plugin roles dispatch as `unityworks-team:<role>`; project agents by their bare name. Tell the user the
roster in one line before dispatching.

## 3. Fan out — in parallel, one message

Dispatch every specialist in a **single message** so they run concurrently. Each prompt contains only:

- the repository absolute paths and their reviews directories;
- the scope: the area (or "all commits not on the default branch, plus uncommitted changes") and the
  changed files relevant to that role;
- the date;
- "File your report per unityworks-team:evidence-report. Return its path(s), verdict and counts."

Do not paste skill content or your own opinions into the prompts; the agents load what they need.

## 4. Triage

When every specialist has returned, confirm each report file exists. Then dispatch
`unityworks-team:tech-lead` with the review directory path(s), the list of report files, and which
directory holds `00-triage.md` (this repository's). If a specialist failed or returned its report
inline, write nothing yourself — tell the tech-lead which role is missing so the gap appears in the
triage.

## 5. Reply

From `00-triage.md`: the verdict; the remediation order as a numbered list (one line each, with
`file:line` and source finding); merged duplicates count; coverage gaps; the triage path.

Stop there. Implementation happens in this session one step at a time, with the user's approval for
each — ask which step to start.
