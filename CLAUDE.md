# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`unityworks-team` is a **Claude Code plugin**, not an application, and it is **project-agnostic**.
[README.md](README.md) lists every command, agent, skill and hook. The original design (written for
the UnityWorks Vision AI pair) is
[2026-09-08-unityworks-agent-team-design.md](2026-09-08-unityworks-agent-team-design.md); its layering
still holds, but project knowledge now lives in the projects.

**The rule that governs every change here: nothing in this repository may name a particular project,
path, framework-specific file or command as a fact.** Project facts belong in that project's
`.claude/team.conf`, `.claude/team/roles/<role>.md`, `.claude/team/invariants.md`,
`.claude/team/checks/`, `.claude/skills/` or `.claude/agents/`. Before committing:

```bash
grep -rniE "vision|kitchen|camera|openapi|alembic|8010|Unityworks_" agents skills commands hooks scripts
```

should only hit generic examples.

Layering (design §2): anything that must happen reliably goes as low as it can — hook/script before
skill, skill before agent. Don't put a deterministic check into an agent prompt.

## Commands

```bash
bash tests/run-tests                                   # hooks + scripts; non-zero on any failure
bash scripts/team-context                              # run inside any project
bash scripts/standup
bash scripts/run-check [name]
claude plugin marketplace update unityworks-local      # after changing the plugin; then plugin update + restart
```

One hook case by hand:

```bash
printf '%s' '{"tool_name":"Bash","cwd":"'"$PWD"'","tool_input":{"command":"git push -f"}}' | bash hooks/destructive-bash; echo $?
```

## Configuration engine

- **`hooks/lib/common`** is sourced by every hook and script: `json_field` (sed, no `jq`),
  `normalise_path` (collapses the doubled backslashes of Windows paths on the wire — without it guards
  silently pass on Windows), `repo_root_of`, and the team.conf readers `conf_values` / `conf_value` /
  `part` / `rest_from` / `under`. The format is line-based on purpose: hooks run on every tool call and
  must not depend on `jq`, Python or Node being installed.
- **Format** (`key = value`, repeat for lists, ` :: ` between parts, `#` comments at line start, CRLF
  tolerated) is specified in `skills/team-setup/SKILL.md`. Adding a key means: reader in the hook or
  script, a row in that skill's table, and tests.
- **Repositories are found by `kind`, never by folder name** (`scripts/find-repo`, which refuses to
  choose between two equal matches). `scripts/team-context` resolves `related` lines into `REPO_<LABEL>`.
- A repository without team.conf must keep working: path hooks do nothing, commands fall back to
  generic behaviour, `/standup` says to run `/team-init`.

## Hooks

Extensionless bash scripts reading tool-call JSON on stdin; exit `2` blocks and returns stderr to
Claude, exit `0` allows. Registered in `hooks/hooks.json` via `${CLAUDE_PLUGIN_ROOT}` and the
`run-hook.cmd` polyglot launcher.

- **Extensionless is load-bearing** — Claude Code on Windows prepends `bash` to commands containing `.sh`.
- **`run-hook.cmd`** uses `!ERRORLEVEL!` with `enabledelayedexpansion`; `%ERRORLEVEL%` expands at parse
  time and made every guard return 0. No bash found → allows (fails open by design).
- **`agent-scope`** identifies agents by `agent_type` in the payload (verified: plugin agents arrive as
  `unityworks-team:<role>`, others by bare name). A settings.json deny-list cannot do this — it would
  bind the main session too.
- **`verify-gate`** (Stop) reads the transcript JSONL; blocks at most once (`stop_hook_active`).
- Command-based hooks resolve the project from the payload's `cwd`; path-based hooks from the file path.
- PreToolUse guards, PostToolUse reminds. Over-blocking gets a guard disabled: keep allow cases next to
  every block case.

## Tests

`tests/run-tests` builds fixture repositories with deliberately unrelated folder names (`api-srv`,
`web-ui`), one with a CRLF team.conf, one without team.conf, and asserts with `expect` (exit code),
`expect_output`, `expect_stdout` and `expect_via_launcher`. Every guard needs: block, allow, no-conf,
Windows-path and launcher cases. Before trusting a new passing test, break the code in a copy and watch
it fail.

## Skills, agents, commands

- Skill `description` states **when to use**, never the procedure.
- Agents carry no Edit tool, begin by reading the project's `CLAUDE.md`, `team.conf` and
  `.claude/team/roles/<role>.md` (which win over the agent's generic lists), and end with the report
  path, verdict and counts.
- Commands call scripts as `${CLAUDE_PLUGIN_ROOT}/scripts/...` (verified to expand); skills refer to
  helpers relative to "this skill's base directory".

`.gitattributes` forces LF — required for the bash scripts and the polyglot launcher.
