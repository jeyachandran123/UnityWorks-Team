---
description: Parallel specialist sweep of the Vision AI pair, then tech-lead triage into one ranked remediation plan
argument-hint: [area]   e.g. auth · contract · vision · frontend · release (default: whole branch)
allowed-tools: Bash, Read, Glob, Agent
---

Area: $ARGUMENTS

## 1. Establish the ground

```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/standup"
date +%F
```

Record the backend and frontend absolute paths, today's date, and the refs under review
(`git rev-parse --short HEAD`, plus "working tree" if uncommitted).

## 2. Choose the roster

| Area | Specialists |
|---|---|
| *(none — whole branch)* | security-engineer, architecture-auditor, devops-engineer, qa-engineer, release-scribe |
| `auth` | security-engineer, backend-engineer, qa-engineer, frontend-engineer |
| `contract` | devops-engineer, backend-engineer, frontend-engineer |
| `vision` | vision-specialist, architecture-auditor, backend-engineer, qa-engineer |
| `frontend` | frontend-engineer, ux-architect, product-analyst, qa-engineer |
| `release` | devops-engineer, release-scribe, product-analyst, security-engineer, architecture-auditor |
| anything else | pick the 3–5 whose failure classes match the area; say which and why |

Tell the user the roster in one line before dispatching.

## 3. Fan out — in parallel, one message

Dispatch every specialist in a **single message** so they run concurrently. Each prompt contains only:

- both repository absolute paths;
- the scope: the area (or "all commits on this branch not on origin/main, plus uncommitted changes")
  and the file list from step 1 relevant to that role;
- the date for `docs/reviews/<date>/`;
- "File your report per unityworks-team:evidence-report. Return its path(s), verdict and counts."

Do not paste skill content or your own opinions into the prompts; the agents load what they need.

## 4. Triage

When every specialist has returned, confirm each report file exists. Then dispatch
`unityworks-team:tech-lead` with the review directory path(s) and the list of report files. If a
specialist failed or returned its report inline, write nothing yourself — tell the tech-lead which
role is missing so the gap appears in the triage.

## 5. Reply

From `00-triage.md`: the verdict; the remediation order as a numbered list (one line each, with
`file:line` and source finding); merged duplicates count; coverage gaps; the triage path.

Stop there. Implementation happens in this session one step at a time, with the user's approval for
each — ask which step to start.
