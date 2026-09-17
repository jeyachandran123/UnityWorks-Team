---
name: invariant-audit
description: Use when asked whether an architectural invariant, boundary, permission rule or "a test enforces this" claim still holds in the UnityWorks Vision AI repositories; before signing off a branch that touches vision_os, compliance, authorization or four-state semantics; or when a guard test may have become a tautology.
---

# Invariant Audit

## Overview

An invariant is held by its test, and a test is only evidence if it **fails when the invariant is
broken**. Re-reading the claim, re-reading the code, or seeing the test pass proves nothing: a test
that passes against both the correct and the broken code is decoration.

**Core rule: no invariant is reported as holding until you have watched its test go red.**

## Procedure — per invariant

1. **Locate** the test by name (see the maps below). Read it. Write down, in one sentence, what it
   actually asserts — often narrower than its name. (`TestSemanticCeiling` scans *identifiers*
   against a fixed vocabulary; a string literal is invisible to it.)
2. **Run it green** on the current tree and keep the real output.
3. **Break the invariant in a scratch copy** of the working tree — never in the repository itself.
   The helper beside this file copies uncommitted changes too, excludes `.git`, and links
   `node_modules` instead of copying it:
   ```bash
   SC="<this skill's base directory>/scratch-copy"
   eval "$(bash "<this skill's base directory>/../../scripts/locate-pair" --print)"   # $BACKEND, $FRONTEND by contents
   COPY="$(bash "$SC" "$BACKEND")"
   echo 'kitchen_alert = True' >> "$COPY/vision_os/perception/__init__.py"   # mutate with Bash, not Edit
   ```
4. **Run it red, in the copy.**
   - Backend: `cd "$COPY" && "<repo>/.venv/Scripts/python.exe" -m pytest <node-id>` — the original
     venv's python, and imports resolve to the copy.
   - Frontend: `cd "$COPY" && npx vitest run tests/<file>`.
   Check the exit code, not the dots: the backend's `addopts = -q` plus your own `-q` hides the
   summary line. If it still passes, the test does not enforce the invariant — **that is a finding.**
5. **Discard:** `bash "$SC" --remove "$COPY"` (refuses any directory it did not create, and unlinks
   `node_modules` before deleting so the original's dependencies are never followed).
6. **Report** with `unityworks-team:evidence-report`: invariant, test node id, the exact mutation,
   green result, red result.

Mutations must be *realistic* — the shape a real regression would take — never a deleted assert.

## Backend invariant map (`unityworks-vision-ai-backend`)

| Invariant | Test | A realistic mutation |
|---|---|---|
| Vision OS migrated verbatim; relative imports only | `tests/app/test_migration.py::TestVisionOsIsUnchanged` | Add `from vision_os.core import x` inside `vision_os/` |
| No `sys.path` manipulation | `tests/app/test_migration.py::TestNoSiblingRepositoryDependency::test_no_source_file_mutates_sys_path` | `sys.path.insert(0, "..")` in an `app/` module |
| No sibling-repo dependency | `tests/app/test_migration.py::TestNoSiblingRepositoryDependency::test_the_platform_imports_in_a_subprocess_with_no_extra_path` | Import something from `../` |
| `app → compliance → vision_os` one way | `tests/compliance/test_boundaries.py::TestTheDependencyRunsOneWay`, `tests/app/test_migration.py::TestApplicationDependencyDirection` | `import compliance` in a `vision_os/` module |
| Semantic ceiling (no business words in platform) | `tests/vision_os/architecture/test_boundaries.py::TestSemanticCeiling`, `tests/compliance/test_boundaries.py::TestNoDomainVocabularyInCode` | Identifier `kitchen_alert = True` in `vision_os/perception/__init__.py` (verified red). A literal `"kitchen"` there stays **green** in both suites — known gap |
| One `AttributeRegistry`, by identity | `tests/app/test_shared_attribute_registry.py` | Build a second registry in `app/vision/composition.py` |
| Three/four-valued compliance | `tests/compliance/test_dataset_regression.py::TestTheSafetyContract::test_not_visible_never_produces_a_violation`, `tests/app/test_observations.py::test_not_visible_survives_the_fold_unchanged`, `tests/app/test_freshness_regression.py::TestThreeValuedSemantics::test_every_declared_attribute_carries_a_refusal_value` | Map `not_visible` → `none` in `app/domain/observations.py` |
| Deny by default (no empty camera tuple) | `tests/app/test_foundation.py -k "no_access_cannot_become"` | Return `()` instead of raising in `AccessDecision.to_grant()` |

## Frontend invariant map (`unityworks-vision-ai-frontend`)

| Invariant | Test | A realistic mutation |
|---|---|---|
| Four states resolved in one place; only `ABSENT` violates | `tests/semantics.test.ts` ("a refused observation cannot produce a violation") | Replace `'not_visible'` with `'absent'` in `src/shared/semantics/observation.ts` (verified: 3 tests red) |
| `—` never `0` | `tests/composition.test.tsx` "a figure refuses a value it does not have" | Default `Figure` value to `0` |
| Meter needs a denominator | `tests/composition.test.tsx`, `tests/art-direction.test.tsx` | Draw with `total = 0` |
| Readiness declaration matches pages | `tests/information-architecture.test.tsx` "marks a destination as awaiting or blocked only where the page says so" | Flip one nav item to `live` |
| One dynamic import | `tests/devtools.test.tsx` (reads `AppRouter?raw`) | Add a second `React.lazy` |
| Unique icon per destination | `tests/icons.test.tsx` | Reuse a `NavIcons` entry |
| Single-flight refresh | `tests/auth.test.tsx` "single-flight refresh" | Call refresh per 401 in `client.ts` |

## Tautology check (guards)

For any permission/guard test: confirm the fixture holds **exactly** the role's permissions. Frontend
role fixtures live in `tests/support.tsx`; backend permissions in `app/authorization/`. An
over-granted fixture makes a "refused" test unreachable — mutate the guard to always admit and watch
whether the test notices.

## Red flags — you are not auditing

- "The test is named `test_semantic_ceiling`, so it holds."
- "I read the code and it's correct."
- "It passed." (Against what broken version?)
- Mutating the test instead of the code.
- Leaving the mutation in the working tree.
