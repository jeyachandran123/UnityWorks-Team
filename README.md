# unityworks-team

The Claude Code plugin for the UnityWorks Vision AI repositories
(`unityworks-vision-ai-backend`, `unityworks-vision-ai-frontend`).

It turns one Claude Code session into a small engineering team: deterministic gates that always run,
procedures that load when a task matches, specialist reviewers you call on, and six commands that
drive them. Design: [2026-09-08-unityworks-agent-team-design.md](2026-09-08-unityworks-agent-team-design.md).

## Install

Both product repos enable the plugin in their committed `.claude/settings.json`, so opening Claude Code
in either repo offers to install it. To install by hand:

    claude plugin marketplace add <path-to>/unityworks-team
    claude plugin install unityworks-team@unityworks-local

After changing anything in this repo: `claude plugin marketplace update unityworks-local` and restart
Claude Code — installed plugins run from a cached copy.

## What you get

### Commands

| Command | Does | Cost |
|---|---|---|
| `/standup` | Git state of both repos, schema pin health, untriaged reviews | a script, no agents |
| `/contract` | Backend export → frontend types → CI pin, reported PASS/FAIL | a script |
| `/review <role> [scope]` | One specialist reviews the diff and files a report | 1 agent |
| `/brief <role> <question>` | One specialist answers one question, no report | 1 agent |
| `/harden [area]` | Parallel specialists → tech-lead triage → one ranked plan | 4–6 agents |
| `/ship` | Both gates, contract, migrations, prod config, docs drift → go/no-go | mostly scripts |

(When another plugin defines the same name, use the qualified form, e.g. `/unityworks-team:harden`.)

### Specialists (`agents/`)

`security-engineer` · `architecture-auditor` · `vision-specialist` · `backend-engineer` ·
`frontend-engineer` · `ux-architect` · `devops-engineer` · `qa-engineer` · `product-analyst` ·
`release-scribe` · `tech-lead`

They advise; they never change code. Each files `docs/reviews/<date>/<role>.md` in the repo it
reviewed; `tech-lead` reads only those reports and writes `00-triage.md`.

### Skills

Plugin (both repos): `contract-sync`, `invariant-audit` (with a `scratch-copy` helper for mutation
testing), `evidence-report`, `prod-readiness`.
Backend `.claude/skills/`: `backend-verify`, `api-route-change`, `vision-os-boundaries`,
`db-migration`, `vision-eval`.
Frontend `.claude/skills/`: `frontend-verify`, `new-route`, `four-states`, `design-system`.

### Hooks (always on)

| Hook | Event | Effect |
|---|---|---|
| `secret-guard` | before Edit/Write | blocks secrets and `.env` writes |
| `protected-tree` | before Edit/Write | blocks edits to the backend's `vision_os/`, `compliance/`, `tools/` |
| `generated-file` | before Edit/Write | blocks hand-edits to the frontend's `openapi.ts` |
| `destructive-bash` | before Bash/PowerShell | blocks `alembic downgrade`, force push, tree discards, `compose down` |
| `agent-scope` | before any tool | keeps specialist agents to reports and read-only commands |
| `contract-reminder` | after Edit/Write | prints the two contract commands after a route edit |
| `verify-gate` | on Stop | once per stop, refuses "done" if edited code never went through its gate |

Path guards identify the repository by its files, so the plugin does nothing in unrelated projects.

## Develop

    bash tests/run-tests          # hook tests (fixture repos, Windows path cases, launcher)

Layout: `.claude-plugin/` manifest + local marketplace · `hooks/` extensionless scripts, `lib/common`,
`run-hook.cmd`, `hooks.json` · `skills/` · `agents/` · `commands/` · `scripts/` (standup,
contract-check, locate-pair) · `tests/`.
