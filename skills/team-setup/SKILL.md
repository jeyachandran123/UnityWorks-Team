---
name: team-setup
description: Use when setting up the unityworks-team plugin for a repository, running /team-init, writing or changing .claude/team.conf, adding a project role brief under .claude/team/roles/, declaring invariants, checks, gates, protected or generated paths, or when a unityworks-team hook, command or agent behaves as if the project is not configured.
---

# Team Setup

## Overview

The plugin knows nothing about any project. Each repository describes itself in files it commits:

| File | Holds | Used by |
|---|---|---|
| `.claude/team.conf` | identity, related repos, gates, checks, protected/generated paths, reminders, blocked commands, project agents, harden rosters | hooks, `/standup` `/check` `/review` `/brief` `/harden` `/ship` |
| `.claude/team/roles/<role>.md` | the project's brief for one plugin agent: what to read, what breaks here, how to prove it | that agent |
| `.claude/team/invariants.md` | invariant → test → realistic mutation | `invariant-audit`, architecture-auditor, qa-engineer |
| `.claude/team/checks/<name>` | deterministic check scripts | `/check`, `/ship` |
| `.claude/skills/`, `.claude/agents/` | project procedures and project-only specialists | Claude Code directly |

Everything is optional. With no `team.conf` the commands still run with generic behaviour, and the
path hooks do nothing.

## team.conf format

One `key = value` per line. Repeat a key for a list. ` :: ` separates the parts of a value. `#`
starts a comment only at the beginning of a line. Paths are relative to the repository root; a
"prefix" matches everything under it (`app/api/` matches `app/api/x.py`).

| Key | Value | Effect |
|---|---|---|
| `name` | display name | shown by `/standup` |
| `kind` | short unique id, e.g. `billing-api` | how *other* repos find this one — never by folder name |
| `reviews` | directory | where agent reports go (default `docs/reviews`) |
| `related` | `<label> :: <kind>` | locates another repo by kind; exposed as `$REPO_<LABEL>` to checks and passed to agents |
| `protect` | `<prefix> :: <why>` | blocks Edit/Write under the prefix, printing why |
| `protect-except` | `<prefix>` | carve-out inside a protected prefix (e.g. `tests/`) |
| `generated` | `<exact path> :: <how to regenerate>` | blocks hand-edits of a generated file |
| `remind` | `<prefix> :: <message>` | printed after an edit under the prefix (follow-up steps) |
| `block` | `<command substring> :: <why / instead>` | blocks a shell command in this repo |
| `gate` | `<prefixes, space-separated> :: <text proving it ran> :: <command>` | on Stop, refuses "done" once if a watched path was edited and the proof text never appeared in a later command |
| `check` | `<name> :: <shell command>` | runnable with `/check <name>`; runs from the repo root with `$REPO_SELF`, `$REPO_<LABEL>`, `$TEAM_SCRIPTS` set; exit 0 = pass |
| `agent` | project agent name | treated as an advisor by the agent-scope hook; offered by `/review` |
| `advisor-block` | `<command substring> :: <why>` | extra commands agents may not run here |
| `advisor-allow` | `<command substring>` | exempts a command from `advisor-block` |
| `harden` | `<area> :: <role> <role> …` | `/harden <area>` roster; area `default` is used with no argument |
| `release-skill` | skill name | loaded by `/ship` for the project's release checklist |

Example:

```
name    = Billing API
kind    = billing-api
related = web :: billing-web

protect        = vendor/ :: Vendored verbatim; update with scripts/vendor.sh instead.
protect-except = vendor/README.md
generated      = src/api/schema.ts :: npm run codegen (after the API exports openapi.json)
remind         = src/routes/ :: Routes changed: npm run export-schema, then /check contract.
block          = prisma migrate reset :: Drops the database. Write a new migration.

gate  = src/ test/ :: npm test :: npm test
gate  = src/ :: npm run lint :: npm run lint
check = contract :: bash .claude/team/checks/contract

harden = default :: security-engineer qa-engineer backend-engineer
```

## /team-init procedure

Work in the repository root. The goal is a `team.conf` in which **every command has been run**.

1. **Read** `CLAUDE.md`, `README*`, CI workflows (`.github/workflows/`, `.gitlab-ci.yml`, …), build
   files (`package.json`, `pyproject.toml`, `Makefile`, `go.mod`, `Cargo.toml`, …) and the top-level
   layout. If there is no `CLAUDE.md`, suggest running `/init` first but continue.
2. **Identity:** propose `name` and a unique `kind`. Look for sibling repositories that already have
   a `team.conf` (`find .. ../.. -maxdepth 3 -path '*/.claude/team.conf'`) and propose `related`
   entries for the ones this repo depends on.
3. **Gates:** take the commands CI runs (lint, typecheck, test, build). For each, **run it** and record
   the exit code and duration. Map each to the paths that should trigger it. Use the exact
   interpreter that works here (a venv path, `npx`, …). A command that fails for environmental
   reasons is still a gate — note the reason for the user.
4. **Guards** — only with evidence, never invented:
   - `generated`: files with a "generated / do not edit" header, or produced by a codegen script.
   - `protect`: vendored or migrated-verbatim trees documented as such.
   - `block`: destructive commands the docs warn about (database resets, `compose down -v`, …).
   - `remind`: documented "after changing X, also do Y" steps.
5. **Checks:** deterministic cross-cutting verifications the docs describe (schema drift, pinned
   versions). Write them as scripts under `.claude/team/checks/` that change no files, and run them.
   Add `.claude/team/checks/** text eol=lf` to the repository's `.gitattributes` — on Windows
   checkouts with `core.autocrlf` a CRLF bash script fails with `$'\r': command not found`.
6. **Invariants:** if the docs declare invariants with tests, write `.claude/team/invariants.md`
   (invariant | test | realistic mutation). Verify each test id exists (collect or grep).
7. **Role briefs:** for the plugin agents relevant to this repo, write short
   `.claude/team/roles/<role>.md` files: *Evidence to read*, *What breaks here*, *How to prove it*.
   Only facts found in the code or docs.
8. **Write** `.claude/team.conf`, then verify it:
   ```bash
   bash "<plugin root>/scripts/team-context"     # identity + related repos resolve
   bash "<plugin root>/scripts/run-check"        # every check runs
   ```
   and pipe a sample payload into each path hook to confirm it fires on a real protected/generated
   path (see the plugin's tests/run-tests for payload shapes).
9. **Report** to the user: the file list, every command run with its result, and anything left
   undecided. Do not commit unless asked.

`<plugin root>` is two directories above this skill's base directory.

## Common mistakes

- Locating another repository by folder name or relative path — use `related` + `kind`.
- A gate whose "proof text" never appears in the command actually run (e.g. proof `pytest` but the
  command is `make test`): the Stop gate then blocks forever-once. Proof text must be a substring of
  the command you expect to run.
- Guards copied from another project without evidence in this one.
- Checks that modify files (they must be safe to run at any time).
