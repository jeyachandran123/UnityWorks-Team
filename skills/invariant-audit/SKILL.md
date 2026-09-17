---
name: invariant-audit
description: Use when asked whether an architectural invariant, boundary, permission rule or "a test enforces this" claim still holds in a repository; before signing off a branch that touches a declared invariant; or when a guard or permission test may have become a tautology.
---

# Invariant Audit

## Overview

An invariant is held by its test, and a test is only evidence if it **fails when the invariant is
broken**. Re-reading the claim, re-reading the code, or seeing the test pass proves nothing: a test
that passes against both the correct and the broken code is decoration.

**Core rule: no invariant is reported as holding until you have watched its test go red.**

## Where the invariants are

1. `.claude/team/invariants.md` in the repository — the project's map: invariant → test → realistic
   mutation. Use it when present.
2. Otherwise build the map yourself from the repository's `CLAUDE.md`, architecture docs and test
   names (grep for "invariant", "boundary", "architecture", "must never"), and say in the report
   that no declared map existed.

## Procedure — per invariant

1. **Locate** the test. Read it. Write down, in one sentence, what it actually asserts — often
   narrower than its name (e.g. a vocabulary test that scans identifiers but not string literals).
2. **Run it green** on the current tree with the project's own test command (its `team.conf` `gate`
   lines or `CLAUDE.md` say how), and keep the real output and exit code.
3. **Break the invariant in a scratch copy** of the working tree — never in the repository itself.
   The helper beside this file copies uncommitted changes too, excludes `.git`, and links
   `node_modules` instead of copying it:
   ```bash
   SC="<this skill's base directory>/scratch-copy"
   COPY="$(bash "$SC" "$(git rev-parse --show-toplevel)")"
   # apply the smallest realistic mutation inside "$COPY" with Bash (sed/echo), not Edit
   ```
4. **Run it red, in the copy.** Python: run the ORIGINAL repository's interpreter from inside the copy
   (e.g. `<repo>/.venv/Scripts/python.exe -m pytest <node-id>`; imports resolve to the copy because
   pytest puts the rootdir first). Node: `npx vitest run <file>` / `npx jest <file>` inside the copy.
   Judge by exit code, not by dots. If it still passes, the test does not enforce the invariant —
   **that is a finding.**
5. **Discard:** `bash "$SC" --remove "$COPY"` (refuses any directory it did not create, and unlinks
   `node_modules` before deleting so the original's dependencies are never followed).
6. **Report** with `unityworks-team:evidence-report`: invariant, test id, the exact mutation, green
   result, red result. Verdict per invariant: enforced / partially enforced (name the gap) / not
   enforced.

Mutations must be *realistic* — the shape a real regression would take — never a deleted assert.

## Tautology check (guards and permissions)

For any permission or guard test, confirm the fixture holds **exactly** the permissions of the role it
represents (the map says where fixtures and the role table live). An over-granted fixture makes a
"refused" test unreachable — mutate the guard to always admit and watch whether the test notices.

## Red flags — you are not auditing

- "The test is named `test_boundaries`, so it holds."
- "I read the code and it's correct."
- "It passed." (Against what broken version?)
- Mutating the test instead of the code.
- Leaving the mutation in the working tree.
