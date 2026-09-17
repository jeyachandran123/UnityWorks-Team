# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`unityworks-team` is a **Claude Code plugin**, not an application. It serves the Vision AI pair in
`../Unityworks_vision_AI/` (`unityworks-vision-ai-backend`, `unityworks-vision-ai-frontend`). The
design is [2026-09-08-unityworks-agent-team-design.md](2026-09-08-unityworks-agent-team-design.md);
[README.md](README.md) lists every command, agent, skill and hook.

The split is deliberate: **roles and cross-repo procedures live here; repo-specific procedures live in
each product repo's `.claude/skills/`**, so they are reviewed in the same PR as the code they
describe. Don't move them here.

Layering (design §2): anything that must happen reliably goes as low as it can — hook/script before
skill, skill before agent. Don't put a deterministic check into an agent prompt.

## Commands

```bash
bash tests/run-tests                                   # hook tests; non-zero on any failure
bash scripts/standup                                   # run from anywhere near the pair
bash scripts/contract-check
claude plugin marketplace update unityworks-local      # after changing the plugin, then restart Claude Code
```

One hook case by hand:

```bash
printf '%s' '{"tool_name":"Bash","tool_input":{"command":"git push -f"}}' | bash hooks/destructive-bash; echo $?
```

## Hooks

Extensionless bash scripts reading tool-call JSON on stdin; exit `2` blocks and returns stderr to
Claude, exit `0` allows. Registered in `hooks/hooks.json` via `${CLAUDE_PLUGIN_ROOT}` and the
`run-hook.cmd` polyglot launcher.

- **Extensionless is load-bearing** — Claude Code on Windows prepends `bash` to commands containing `.sh`.
- **`run-hook.cmd`** uses `!ERRORLEVEL!` with `enabledelayedexpansion`; `%ERRORLEVEL%` expands at parse
  time and made every guard return 0. No bash found → allows (fails open by design).
- **`hooks/lib/common`** is sourced by every hook: `json_field` (sed, no `jq`), `normalise_path`
  (collapses the doubled backslashes of Windows paths on the wire — without it guards silently pass on
  Windows), `repo_root_of` and the `is_vision_backend` / `is_vision_frontend` marker checks.
- **Guards scope by repository markers, never by bare path fragments**, so the plugin is inert in
  unrelated projects. A new path guard must do the same and needs: block case, allow case,
  unrelated-repo case, Windows-backslash case, launcher case.
- **`agent-scope`** restricts only this plugin's agents, identified by `agent_type` in the payload.
  A settings.json deny-list cannot do this — it would bind the main session too. Writes are allowed
  only under `docs/reviews/` or inside a `scratch-copy` (marker `.unityworks-scratch-copy`).
- **`verify-gate`** (Stop) reads the transcript JSONL, finds Edit/Write calls into gated paths and
  checks a gate command appears on a later line; it blocks at most once (`stop_hook_active`).
- PreToolUse guards, PostToolUse reminds. Never hook what a product-repo test already enforces.
- Over-blocking gets a guard disabled: keep negative cases next to every pattern.

## Tests

`tests/run-tests` builds fixture repos in a temp dir (backend, frontend, an unrelated repo, a scratch
copy) and asserts with `expect` (exit code), `expect_output` (substring, or empty = silent) and
`expect_via_launcher`. Before trusting a new passing test, break the hook in a copy and watch it fail.

## Skills, agents, commands

- Skill frontmatter `description` states **when to use**, never the procedure. Every command, path and
  test id a skill cites was run against the real repos; keep it that way when editing.
- Agents carry no Edit tool, load skills by name, and end with the report path, verdict and counts.
- **Never locate a repository by folder name or fixed path.** `scripts/find-repo <kind>` identifies
  one by its contents (and refuses to guess between two); `scripts/locate-pair` wraps it for the
  Vision pair and exports `UWV_SCHEMA_PATH` so the frontend generator no longer needs a sibling named
  `unityworks-vision-ai-backend`. Commands call them via `${CLAUDE_PLUGIN_ROOT}/scripts/...`; skills via
  `<this skill's base directory>/../../scripts/...`; agents follow the "Locating repositories" rule
  in their Artifact section. A new repository kind is a new case in `find-repo` plus a test.
- On this machine bare `python` is not on PATH; the backend uses `.venv/Scripts/python.exe -m ...`.

`.gitattributes` forces LF — required for the bash scripts and the polyglot launcher.
