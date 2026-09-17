# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`unityworks-team` is a **Claude Code plugin**, not an application. It holds what is identical across
`../unityworks-vision-ai-backend` and `../unityworks-vision-ai-frontend`: shared hook scripts today,
and (per the design) cross-repo specialist agents, skills and slash commands later. Procedures that
are specific to one repo live in that repo's own `.claude/skills/` so they are reviewed in the same
PR as the code they describe — don't move them here.

The authoritative design is
[2026-09-08-unityworks-agent-team-design.md](2026-09-08-unityworks-agent-team-design.md).
Its build order is: Phase 0 hooks + CI → 1 skills → 2 agents → 3 commands → 4 `/harden`. Only the
Phase 0 hook scripts exist so far; `agents/`, `skills/`, `commands/` and `hooks/hooks.json` are
planned but not yet present.

## Commands

```bash
bash tests/run-tests          # all hook tests; exits non-zero on any failure
claude plugin marketplace add ./
claude plugin install unityworks-team@unityworks-local
```

There is no build, lint or single-test runner. To run one case, pipe a payload straight into the hook
and check the exit code:

```bash
printf '%s' '{"tool_name":"Bash","tool_input":{"command":"git push -f"}}' | bash hooks/destructive-bash; echo $?
```

## Hook architecture

Each hook in `hooks/` is an **extensionless** bash script that reads the tool-call JSON on stdin.
Exit `2` blocks the call and returns stderr to Claude; exit `0` allows it.

- **Extensionless is load-bearing.** Claude Code on Windows prepends `bash` to any command containing
  `.sh`, which breaks direct invocation. Don't add extensions.
- **`run-hook.cmd` is a cmd/bash polyglot launcher** that product repos invoke as
  `run-hook.cmd <script>`. On Windows the batch half finds Git Bash; on Unix `:` is a no-op and it
  falls through to `exec bash`. Inside the batch `if (...)` blocks it must use `!ERRORLEVEL!` with
  `enabledelayedexpansion` — `%ERRORLEVEL%` is expanded at parse time and made every guard return 0
  (commit 8a667e3). If no bash exists it deliberately allows the action.
- **No `jq`.** Fields are pulled from raw JSON with `sed`. Because the JSON is never unescaped,
  Windows paths arrive with doubled backslashes (`C:\\Users\\...`); path-matching hooks normalise
  with `path="${path//\\\\//}"` before their forward-slash `case` patterns. Any new path-matching
  hook needs the same line, plus a Windows-backslash test case, or it silently passes on Windows.
- **`PreToolUse` guards, `PostToolUse` reminds.** Only `PreToolUse` can prevent an action, so
  `secret-guard`, `protected-tree`, `generated-file` and `destructive-bash` are pre-hooks;
  `contract-reminder` is post-edit and advisory (always exits 0).
- **Never hook what a test already enforces** in the product repos (observation states, dependency
  arrow, etc.). Hooks cover facts about the *process* that no test can observe.
- **Over-blocking gets a guard disabled.** `secret-guard` requires a secret-shaped name that *starts*
  an identifier (`\b`) so `expected_password_hash` / `BUILD_TOKEN` pass; keep negative test cases
  alongside every new pattern.

What each guard protects, briefly: `protected-tree` blocks edits under `vision_os/`, `compliance/`,
`tools/` (migrated verbatim; `*/tests/*` exempt); `generated-file` blocks hand-edits of the frontend's
`src/shared/types/openapi.ts`; `contract-reminder` fires on `*/app/api/*` edits to prompt
`export_openapi.py` + `npm run types:generate`; `destructive-bash` blocks `alembic downgrade`, force
pushes and working-tree discards.

## Tests

`tests/run-tests` has three helpers: `expect` (exit code), `expect_output` (substring of
stdout+stderr, or empty for "must print nothing") and `expect_via_launcher` (drives the hook through
`run-hook.cmd`'s Unix half to prove dispatch and exit-code propagation). Every guard should have a
block case, an allow case, a Windows-backslash-path case where paths matter, and a launcher case.

`.gitattributes` forces LF line endings — required for the bash scripts and the polyglot launcher.
