# unityworks-team

A Claude Code plugin that turns one session into a small engineering team — for **any** repository.

The plugin knows nothing about a particular project. Each repository describes itself in files it
commits (`.claude/team.conf` and friends); the plugin's hooks, commands and agents read them.

## Install

    claude plugin marketplace add <path-to>/unityworks-team      # or the GitHub repo
    claude plugin install unityworks-team@unityworks-local --scope project   # run inside the project

A project can also enable it for everyone who clones it, in its committed `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": { "unityworks-local": { "source": { "source": "github", "repo": "jeyachandran123/UnityWorks-Team" } } },
  "enabledPlugins": { "unityworks-team@unityworks-local": true }
}
```

After changing the plugin: `claude plugin marketplace update unityworks-local`, then
`claude plugin update unityworks-team@unityworks-local --scope project` in each project, and restart
Claude Code — installed plugins run from a cached copy.

## Set up a project

In the project, run **`/team-init`**. Claude reads the repository, runs its lint/test/build commands,
and writes a `.claude/team.conf` in which every command has actually been run. Review it and commit it.
Format and procedure: [skills/team-setup/SKILL.md](skills/team-setup/SKILL.md).

| Project file | Holds |
|---|---|
| `.claude/team.conf` | identity (`kind`), related repos, gates, checks, protected/generated paths, reminders, blocked commands, project agents, `/harden` rosters |
| `.claude/team/roles/<role>.md` | the project's brief for a plugin agent: what to read, what breaks here |
| `.claude/team/invariants.md` | invariant → test → realistic mutation |
| `.claude/team/checks/<name>` | check scripts run by `/check` |
| `.claude/skills/`, `.claude/agents/` | project procedures and project-only specialists |

Repositories find each other by `kind`, never by folder name or location.

## What you get

### Commands

| Command | Does | Cost |
|---|---|---|
| `/team-init` | Configure this repository for the team | one session |
| `/standup` | Git state of this and related repos, review status, configuration | a script |
| `/check [name]` | Run the project's declared checks, PASS/FAIL | scripts |
| `/review <role> [scope]` | One specialist reviews the changes and files a report | 1 agent |
| `/brief <role> <question>` | One specialist answers one question, no report | 1 agent |
| `/harden [area]` | Parallel specialists → tech-lead triage → one ranked plan | several agents |
| `/ship` | Gates + checks + the project's release checklist → go/no-go | mostly scripts |

### Specialists (`agents/`)

`security-engineer` · `architecture-auditor` · `backend-engineer` · `frontend-engineer` ·
`ux-architect` · `devops-engineer` · `qa-engineer` · `product-analyst` · `release-scribe` · `tech-lead`

Generic by design; each reads the project's `CLAUDE.md`, `team.conf` and its role brief before working.
They advise and never change code: reports go to `<reviews>/<date>/<role>.md` in the repo reviewed.
Projects add their own specialists in `.claude/agents/` and list them as `agent = <name>`.

### Skills

`evidence-report` (the report shape) · `invariant-audit` (prove a test enforces an invariant by
mutating a scratch copy) · `team-setup` (the configuration format and `/team-init` procedure).

### Hooks (always on, driven by team.conf)

| Hook | Event | Effect |
|---|---|---|
| `secret-guard` | before Edit/Write | blocks secrets and `.env` writes — every repository |
| `protected-tree` | before Edit/Write | blocks edits under `protect` prefixes |
| `generated-file` | before Edit/Write | blocks hand-edits of `generated` files |
| `agent-scope` | before any tool | keeps specialist agents to report writes and read-only commands |
| `destructive-bash` | before Bash/PowerShell | force push and tree discards everywhere; `block` commands per project |
| `path-reminder` | after Edit/Write | prints `remind` messages |
| `verify-gate` | on Stop | refuses "done" once if an edited path's `gate` never ran afterwards |

A repository without `team.conf` gets only the universal protections.

## Develop

    bash tests/run-tests          # hooks and scripts against fixture repositories

Layout: `.claude-plugin/` manifest + local marketplace · `hooks/` extensionless scripts, `lib/common`,
`run-hook.cmd`, `hooks.json` · `scripts/` (find-repo, team-context, run-check, standup) · `skills/` ·
`agents/` · `commands/` · `tests/`.
