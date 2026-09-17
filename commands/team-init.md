---
description: Set up the unityworks-team plugin for this repository — analyse it and write a verified .claude/team.conf (plus optional role briefs, invariants and checks)
argument-hint: [notes, e.g. "the API repo is ../billing-api"]
allowed-tools: Bash, Read, Grep, Glob, Write, Edit, Skill
---

Notes from the user: $ARGUMENTS

Load the `unityworks-team:team-setup` skill and follow its **/team-init procedure** exactly, in the
root of the current git repository (`git rev-parse --show-toplevel`).

- If `.claude/team.conf` already exists, read it first and propose changes as a diff against it instead
  of rewriting it; apply only what the user approves.
- Every command you put in a `gate` or `check` line must have been run here, with its exit code
  reported. A guard (`protect`, `generated`, `block`, `remind`) needs evidence from the code or docs.
- Verify at the end with the plugin's scripts:
  ```bash
  bash "${CLAUDE_PLUGIN_ROOT}/scripts/team-context"
  bash "${CLAUDE_PLUGIN_ROOT}/scripts/run-check"
  bash "${CLAUDE_PLUGIN_ROOT}/scripts/standup"
  ```
- Do not commit. Finish with: files written, each command run and its result, and open questions.
