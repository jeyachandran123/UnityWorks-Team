---
name: contract-sync
description: Use when a backend route, request or response model, or Pydantic schema in unityworks-vision-ai-backend changes; when the frontend's openapi.ts looks wrong or out of date; when types:check or export_openapi.py --check fails; or when frontend CI fails but npm run verify passes locally.
---

# Contract Sync

## Overview

The frontend is typed from a **committed** schema export, never a live endpoint (`/openapi.json`
mounts only under `APP_DEBUG=true`). The chain is:

```
backend app/api/** + models  →  scripts/export_openapi.py  →  docs/api/openapi.json (committed)
   →  frontend scripts/generate-types.mjs  →  src/shared/types/openapi.ts (committed, generated)
```

A change is not done when the backend is green. It is done when **both halves are regenerated and
both gates pass**. No test inside either repo can see the seam; only this procedure and CI can.

## Procedure

Run from `Unityworks_vision_AI/`. On this machine bare `python` is not on PATH — use the venv.

```bash
# 1. Backend: re-export and confirm
cd unityworks-vision-ai-backend
.venv/Scripts/python.exe scripts/export_openapi.py
.venv/Scripts/python.exe scripts/export_openapi.py --check     # must now pass

# 2. Frontend: regenerate from the sibling export
cd ../unityworks-vision-ai-frontend
npm run types:generate
npm run verify          # types:check → typecheck → lint → test → build

# 3. Look at what actually moved
git -C ../unityworks-vision-ai-backend diff --stat docs/api/openapi.json
git diff --stat src/shared/types/openapi.ts
```

Then fix every TypeScript error `typecheck` reports **in the consuming code** (`src/shared/api/*.ts`,
features). Never edit `openapi.ts` by hand — the `generated-file` hook blocks it, and the edit would
vanish at the next generate anyway.

## The CI pin (frontend)

Frontend CI checks out only itself, fetches the schema from
`raw.githubusercontent.com/jeyachandran123/unityworks-vision-ai-backend/<sha>/docs/api/openapi.json`
using the SHA in `.github/backend-schema.sha`, and points the generator at it via `UWV_SCHEMA_PATH`.

| Symptom | Cause | Fix |
|---|---|---|
| `verify` green locally, `types:check` red in frontend CI | Pin names an older schema than the one `openapi.ts` was generated from | Set the pin to the backend commit that holds the current export |
| CI `curl` 404 | Pinned SHA was never pushed | Push the backend first; the pin must be a **pushed** commit |
| Backend CI `export_openapi.py --check` red | Route changed, export not re-run | Step 1 above |

Reproduce frontend CI exactly before bumping anything:

```bash
SHA="$(tr -d '[:space:]' < .github/backend-schema.sha)"
F="$(mktemp -d)/schema.json"
git -C ../unityworks-vision-ai-backend show "$SHA:docs/api/openapi.json" > "$F"
UWV_SCHEMA_PATH="$(cygpath -w "$F" 2>/dev/null || echo "$F")" npm run types:check   # node on Windows cannot read /tmp paths
```

Exit 1 with "openapi.ts is out of date" means the pin is stale. Repeat with `HEAD` in place of
`$SHA` to confirm the current backend export is the one the committed types match.

Bump order is fixed: commit backend export → push backend → write that SHA to
`.github/backend-schema.sha` → commit frontend types **and** pin together.

## When a type moves or is renamed

1. Grep the frontend for the old name before regenerating: `grep -rn "OldName" src tests`.
2. Regenerate, then let `typecheck` enumerate every consumer — do not hunt by hand.
3. Test stubs in `tests/support.tsx` must match the new shape, or tests pass against a response the
   backend no longer sends.
4. A removed field that the UI rendered: render `—` with a reason, never a fabricated default.

## Common mistakes

- Running `types:generate` before `export_openapi.py` — regenerates the old contract, looks clean.
- Committing frontend types without bumping the pin — local green, CI red.
- Pinning an unpushed commit.
- "Fixing" `types:check` by editing `openapi.ts`.
