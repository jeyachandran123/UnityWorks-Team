---
description: Go/no-go for releasing the Vision AI pair — both gates, contract, migrations, production config and docs drift
allowed-tools: Bash, Read, Grep, Glob, Skill, Write
---

Load the `unityworks-team:prod-readiness` skill and execute every section of it, in order, from the
directory that holds both repositories. Notes: $ARGUMENTS

Deterministic parts first, so a red gate is known before anything slow runs:

```bash
bash "${CLAUDE_PLUGIN_ROOT}/scripts/standup"
bash "${CLAUDE_PLUGIN_ROOT}/scripts/contract-check"
```

Then the gates, backend and frontend in parallel as background commands (the backend suite takes
minutes):

- backend: `ruff check app tests`, `black --check app tests`, `pytest` — with the repo's venv python
- frontend: `npm run verify`

While they run, do the database and production-configuration sections (read-only: `alembic heads`,
settings and docs — never `.env`, never a real database).

Rules:

- Every row gets PASS, FAIL or **NOT RUN (reason)**. NOT RUN on a blocker row means no-go.
- CI status is **unverified** unless shown (`gh` is not installed).
- Invariants: a green full suite is recorded; only mutate-test (per `invariant-audit`) if the user
  asked for a deep check in the notes above.
- Change nothing. No fixes, no pin bump, no commits.

Write the verdict file per the skill, then reply with: **GO / GO AFTER BLOCKERS / NO-GO**, the table of
rows, each blocker in one line with its fix command, and the report path.
