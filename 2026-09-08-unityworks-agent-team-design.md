# UnityWorks Agent Team — Design

| | |
|---|---|
| **Date** | 8 September 2026 |
| **Scope** | `unityworks-vision-ai-backend`, `unityworks-vision-ai-frontend`, and a new `unityworks-team` plugin repo |
| **Status** | Approved design — implementation plan to follow |
| **Author** | Jeyachandran S, with Claude Code |

---

## 1. Problem

Two production repositories are being pushed to release on branches named
`feat/unityworks-vision-os-prod-hardining` (backend) and `…-hardening` (frontend). Both carry
strong, test-enforced architectural invariants. Neither has continuous integration. The contract
between them — an OpenAPI schema exported from the backend and compiled into frontend types — is
maintained by hand, across two directories, with no gate.

The request is for a structured team of specialists that makes solo development feel like directing
a staffed engineering group, without inventing ceremony that does not survive contact with the code.

### What is actually true today

| Fact | Evidence |
|---|---|
| Backend has 8 uncommitted modified files, all on the security surface | `app/auth/tokens.py`, `app/auth/service.py`, `app/authorization/platform.py`, `app/domain/audit.py`, `app/api/dependencies.py`, `app/api/platform.py`, `app/api/routes.py`, `app/vision/manager.py` |
| Backend is 9 commits ahead of `origin/main` | `git log origin/main..HEAD` |
| Neither repo has CI | no `.github/` in either tree |
| `gh` CLI is not installed | `gh: command not found` |
| The backend README is stale and now inaccurate | claims "there is no restaurant, camera, incident, notification or report route"; all exist |
| The type generator cannot run in CI | `scripts/generate-types.mjs` hard-codes `../unityworks-vision-ai-backend/docs/api/openapi.json`, a sibling-directory read |
| `.env` is correctly gitignored and untracked | `.gitignore:2-4`; `git ls-files` finds nothing |

---

## 2. Constraints and decisions

Four decisions were taken before design, and everything below follows from them.

| # | Decision | Consequence |
|---|---|---|
| D1 | **Advisors, not autonomous builders.** Specialists analyse, review and plan; implementation happens in the main session under human approval. | Agents carry no source-editing tools. Their only write access is a narrow reports window. |
| D2 | **Hybrid packaging.** Role definitions live once in a shared plugin; repo procedures are committed inside each repo. | Roles cannot drift between repos. Procedures travel with `git clone` and are reviewed alongside the code they describe. |
| D3 | **First slice is shipping prod-hardening**, not building an org chart. | Phase ordering is driven by release risk, and the security surface goes first. |
| D4 | **Contract seam solved by env override** (`UWV_SCHEMA_PATH`), CI fetching the schema at a pinned commit. | ~10 lines changed in the generator; local developer ergonomics unchanged. |

### The organising principle

Anything that must happen **reliably** belongs as low in this stack as it can go:

| Layer | Mechanism | Runs when | Token cost |
|---|---|---|---|
| 1 · Gates | hooks, `verify`, CI | always, deterministically | ~0 |
| 2 · Procedures | skills | when the task matches | small |
| 3 · Judgment | subagents | when invoked | large — each cold-starts |
| 4 · Entry points | slash commands | when typed | — |

The characteristic failure of a large agent roster is placing layer-1 knowledge in layer 3. A hook
that blocks on an unregenerated contract is worth more than a specialist who *usually remembers* to
check.

### Two structural honesties this design accepts

**Subagents are not persistent colleagues.** Each spawn cold-starts and forgets on completion.
Continuity lives in files — CLAUDE.md, skills, specs, finding reports — never in an agent.

**Read-only is a guardrail, not a sandbox.** Withholding `Edit`/`Write` is airtight, but agents need
`Bash` (a QA specialist that cannot run pytest is useless), and `Bash` can mutate. Enforcement is
therefore layered: no source-editing tools in frontmatter, plus a command deny-list in each repo's
`.claude/settings.json`. This stops a distracted agent, which is the actual threat, not a determined
adversary.

---

## 3. Layout

```
ProjectsAndDocs/atlas/
├── unityworks-team/                       NEW — small, its own git repo
│   ├── .claude-plugin/
│   │   ├── plugin.json
│   │   └── marketplace.json               local marketplace for `claude plugin install`
│   ├── agents/                            11 specialists, cross-repo
│   ├── skills/                            4 cross-repo procedures
│   ├── commands/                          6 front doors
│   └── hooks/
│       ├── hooks.json
│       ├── run-hook.cmd                   Windows polyglot wrapper
│       └── <extensionless bash scripts>
│
├── unityworks-vision-ai-backend/
│   ├── .claude/
│   │   ├── settings.json                  repo hooks + permission allow/deny
│   │   └── skills/                        5 backend procedures
│   ├── .github/workflows/ci.yml           NEW
│   └── docs/reviews/<date>/               NEW — finding reports
│
└── unityworks-vision-ai-frontend/
    ├── .claude/
    │   ├── settings.json
    │   └── skills/                        4 frontend procedures
    ├── .github/workflows/ci.yml           NEW
    └── docs/reviews/<date>/               NEW
```

**Why the split falls here.** A role ("what does a world-class CV specialist look for?") is identical
in both repos and should exist once. A procedure ("how do I prove the semantic ceiling still holds?")
is repo-specific, changes when the code changes, and must be reviewable in the same pull request as
the code it describes.

Plugin mechanics are copied from the verified structure of the installed `superpowers` plugin:
`.claude-plugin/plugin.json` and `marketplace.json`; `hooks/hooks.json` shaped as
`{"hooks": {"<Event>": [{"matcher": …, "hooks": [{"type": "command", "command": "…", "shell": "bash"}]}]}}`;
`${CLAUDE_PLUGIN_ROOT}` for paths.

---

## 4. Layer 1 — Gates

**Governing rule: never hook what a test already enforces.** Both repos already test the four
observation states, the one-way dependency arrow, the semantic ceiling, single-flight refresh, the
single dynamic import and icon uniqueness. Duplicating those in hooks creates two sources of truth
that will disagree. Hooks receive only what no test can observe — facts about the *process*.

### 4.1 Hooks

| Repo | Event | Matcher | Behaviour |
|---|---|---|---|
| backend | `PostToolUse` | Edit/Write on `app/api/**` | Remind: route changed ⇒ `python scripts/export_openapi.py` ⇒ frontend `npm run types:generate` |
| backend | `PreToolUse` | Edit/Write on `vision_os/**`, `compliance/**`, `tools/**` | Require explicit confirmation — these trees are migrated verbatim |
| backend | `PreToolUse` | Bash matching `alembic downgrade`, `git push --force`, `git checkout -- .` | Block |
| backend | `Stop` | — | If tracked files changed but `ruff`/`black`/`pytest` never ran this session, say so |
| frontend | `PreToolUse` | Edit/Write on `src/shared/types/openapi.ts` | Block — the file is generated; the generator is its only legitimate author |
| frontend | `Stop` | — | If `src/**` changed and `npm run verify` never ran, say so |
| both | `PreToolUse` | Edit/Write | Block writes containing key-shaped strings; block writes to `.env` |

Rationale for the two `Stop` hooks: `verification-before-completion` is a rule I am asked to follow
by judgment. Mechanising it removes the judgment.

Event choice is not cosmetic. `PreToolUse` is the only event that can *prevent* an action;
`PostToolUse` runs after the tool has already succeeded and can therefore only inform. Guards that
must hold (generated files, secrets, destructive commands) are `PreToolUse`; reminders about
follow-up work in another repo are `PostToolUse`.

**Windows constraint.** Hook scripts are extensionless and invoked through a `run-hook.cmd` polyglot
wrapper. Claude Code on Windows prepends `bash` to any command containing `.sh`, which breaks direct
invocation. This is the pattern `superpowers` uses and it is copied deliberately.

### 4.2 Continuous integration

Two workflows, mirroring the gates each CLAUDE.md already declares.

**Backend** — Python 3.11 → `pip install -e ".[inference,test]"` → `ruff check app tests` →
`black --check app tests` → `pytest` → `python scripts/export_openapi.py --check`.

**Frontend** — Node 20 → `npm ci` → `npm run verify`.

Coverage is *not* added to backend `addopts`. `tests/vision_os/conftest.py` skips timing-budget tests
when `sys.gettrace()` is set, so a default `--cov` silently deletes that coverage rather than
measuring it.

### 4.3 The contract seam (decision D4)

`npm run verify` includes `types:check`, which today reads a sibling directory absent in CI.

Change `scripts/generate-types.mjs` to resolve its schema as:

```
process.env.UWV_SCHEMA_PATH  ??  <existing sibling path>
```

Local development is unaffected. Frontend CI fetches `docs/api/openapi.json` from the backend
repository at a **pinned commit** recorded in the frontend repo, making the contract version an
explicit, reviewable fact rather than whatever happened to be on disk.

Rejected alternatives: mirroring the schema into the frontend repo (produces perpetual contract-bump
pull requests); checking out both repos side by side in CI (needs a token for private repos and
couples the two CI runs).

---

## 5. Layer 2 — Skills

### Plugin skills — true for both repos

| Skill | Encodes |
|---|---|
| `contract-sync` | The OpenAPI seam end to end: export → generate → verify, and what to do when a type moves |
| `invariant-audit` | How to audit any declared invariant: locate its test, **prove the test fails when the invariant is broken**, then report. Auditing a claim by re-reading the claim is theatre |
| `evidence-report` | The output contract every specialist obeys (§7). Findings from eleven agents are comparable only if identically shaped |
| `prod-readiness` | The release checklist specific to these two repos |

### `unityworks-vision-ai-backend/.claude/skills/`

`backend-verify` (ruff/black/pytest scoping; why coverage is never default) · `api-route-change`
(router → schema → permission → test → `export_openapi` → notify frontend, as one ritual) ·
`vision-os-boundaries` (which test proves the arrow, the ceiling, the forbidden ports) ·
`db-migration` (alembic, observation partitions, downgrade danger) · `vision-eval`
(`tools/vision_eval`, datasets, what a regression means).

### `unityworks-vision-ai-frontend/.claude/skills/`

`frontend-verify` (the chain and how to read each failure) · `new-route` (nav model + router +
permission + readiness + the four structural suites that must move together) · `four-states`
(`PRESENT/ABSENT/NOT_VISIBLE/UNKNOWN`; `—` never `0`; `available` never `records.length`) ·
`design-system` (compose from `product.tsx`; icons only from `icons.tsx`).

---

## 6. Layer 3 — The roster

Each specialist is specified by three things, never by a job title: **the evidence it reads**, **the
failure class it catches**, **the artifact it produces**.

| Agent | Reads | Catches | Model |
|---|---|---|---|
| `product-analyst` | `docs/architecture/**`, `navigation.ts`, `readiness` declarations, `capabilities.ts` | Claimed readiness a page cannot back; permission↔feature gaps; documentation that lies | opus |
| `ux-architect` | `shared/ui/**`, art-direction / composition / icons suites, feature pages | Fabricated `0`s, colour as the only signal, pages opting out of `PageIntro`, missing accessible names | opus |
| `frontend-engineer` | `src/**`; runs `verify` | `fetch` outside `client.ts`, token-storage violations, a second dynamic import, tautological fixtures | sonnet |
| `backend-engineer` | `app/**`; runs pytest subsets | Logic leaking into `app/vision/` composition, route↔permission mismatch, analysis-loop thread safety | sonnet |
| `vision-specialist` | `vision_os/**`, `compliance/**`, `config/policies\|rules`, `tools/vision_eval`, `datasets/` | Semantic-ceiling breaches, three-valued collapse, detector/eval regressions, `AttributeRegistry` identity | opus |
| `devops-engineer` | `pyproject.toml`, `alembic.ini`, `migrations/`, `docs/deployment/`, workflows, env handling | Missing CI, production start guards, migration risk, `SERVE_FRAMES`/`ALLOW_EVIDENCE` defaults | sonnet |
| `security-engineer` | `app/auth/**`, `authorization/**`, `domain/audit.py`, cookie flags, evidence paths | Refresh rotation, per-route enforcement, PII/evidence exposure, deny-by-default holes | opus |
| `qa-engineer` | `tests/**` in both repos | Coverage gaps, over-granting fixtures that turn guard tests into tautologies, flake sources | sonnet |
| `architecture-auditor` | boundary and conformance suites | The six invariants, one at a time, each proven by running its test | opus |
| `release-scribe` | READMEs, `docs/**`, CHANGELOG, git log | Documentation drift against actually routed behaviour | sonnet |
| `tech-lead` | **`docs/reviews/<date>/*.md` only — not the source tree** | Duplicate and contradictory findings; unranked severity; wrong remediation order | opus |

### On `tech-lead`

The cold-start objection to a triage agent is answered by narrowing its input. It reads finding
reports, not the repository. Its context is therefore bounded and cheap, it produces a durable
artifact, and its lack of prior involvement becomes an advantage: a triager who was not in the weeds
is not anchored to any one specialist's framing.

Output: `docs/reviews/<date>/00-triage.md` — deduplicated findings, ranked by severity, ordered by
remediation dependency.

### Tool grants

All eleven agents: `Read`, `Grep`, `Glob`, `Bash`, `Write`. No `Edit`, no `NotebookEdit`.

`Write` is admitted **only** so a specialist can file its report. Each repo's `.claude/settings.json`
permits `Write` under `docs/reviews/**` and denies it elsewhere, and denies mutating Bash commands
(`git commit|push|checkout`, `rm`, `alembic downgrade`, `npm install`, output redirection).

*Implementation risk:* path-scoped write permissions must be verified behaving as specified during
Phase 2. If they do not, the fallback is that agents return findings as their final message and the
main session persists them — costlier in context, identical in outcome.

---

## 7. The report contract

Every specialist writes `docs/reviews/<date>/<role>.md`. `tech-lead` consumes them. Uniform shape is
what makes eleven independent outputs comparable.

```markdown
---
role: security-engineer
repo: unityworks-vision-ai-backend
ref: <git sha or "working tree">
date: 2026-09-08
---

## Finding 1 — <one sentence claim>
- **Severity:** blocker | major | minor | note
- **Location:** app/auth/tokens.py:88
- **Evidence:** <command run, and its real output>
- **Failure scenario:** <concrete inputs → wrong behaviour>
- **Recommendation:** <what to change>
```

Rules: no finding without a `file:line`; no finding without evidence that was actually executed; a
verified-clean area is reported as clean rather than omitted, so silence is never ambiguous.

---

## 8. Layer 4 — Commands

| Command | Does | Cost |
|---|---|---|
| `/standup` | Git state of both repos, uncommitted work, branch divergence, unaddressed findings | no agents |
| `/contract` | Runs the OpenAPI seam and reports drift | deterministic |
| `/review <role>` | One specialist over the current diff | 1 agent |
| `/brief <role>` | One question to one specialist, no full review | 1 agent |
| `/harden [area]` | Parallel specialist fan-out → `tech-lead` triage → one ranked plan | many agents |
| `/ship` | Both verify chains + contract + docs drift + migration status → go/no-go | mostly deterministic |

Six is a deliberate ceiling. Commands that go untyped are clutter that still costs context.

---

## 9. Workers

There are no persistent daemons. Four real mechanisms, used for what each is good at:

| Mechanism | Used for | Availability |
|---|---|---|
| Background Bash | The `vision_os` performance suite; frontend tests with 25s timeouts | now |
| Parallel subagent fan-out | The `/harden` sweep — this is what actually feels like a team | now |
| `/loop` | Watching a long CI run to completion | now, session-bound |
| Scheduled cloud agents | Nightly contract-drift check; weekly dependency audit | **last**, requires pushed branches |

Scheduled agents are sequenced last on purpose: they run against pushed refs, and there is currently
uncommitted and unpushed work in the backend.

---

## 10. First slice — `/harden` for prod-hardening

```
1. security-engineer      the 8 uncommitted files: auth, tokens, authorization, audit
2. architecture-auditor   prove the six invariants still hold on this branch
3. devops-engineer        the two missing CI workflows + UWV_SCHEMA_PATH
4. qa-engineer            coverage and fixture honesty across the 9 new commits
5. release-scribe         the README that now lies about routed behaviour
        ↓  parallel, each writing docs/reviews/<date>/
6. tech-lead              dedupe, rank, order → 00-triage.md
7. main session           implement against the triage, human approval per step
```

Security leads because uncommitted changes to token and authorization code are the highest-risk
artefact in either repository today.

---

## 11. Build order

| Phase | Delivers | Rationale |
|---|---|---|
| 0 | Hooks, two CI workflows, `UWV_SCHEMA_PATH` | Determinism first; everything above rests on it |
| 1 | 4 plugin + 5 backend + 4 frontend skills | Agents must have procedures to load before they exist |
| 2 | 11 agent definitions + permission wiring | Verify path-scoped writes here |
| 3 | 6 commands | Front doors last — they compose what is already proven |
| 4 | Run `/harden` for real | The system's first genuine exercise |

Each layer is exercised by the one above before anything depends on it.

---

## 12. Out of scope

Deliberately excluded, and why:

- **A contract-steward agent.** The check is deterministic; a hook and CI do it faster and more
  reliably.
- **Hooks duplicating existing tests.** Two sources of truth for an invariant is worse than one.
- **Autonomous committing or PR-opening agents.** Excluded by D1; `gh` is not installed and CI does
  not yet exist, so the prerequisites are absent regardless.
- **The other four workspace projects** (`backend/`, `frontend/`, `vision_os_demo/`,
  `vision_os_validation_console/`). The plugin will work anywhere, but no procedures are written for
  them in this phase.
- **Per-agent memory or a shared agent knowledge base.** Files are the memory. Anything else is a
  second, unversioned source of truth.

---

## 13. Open risks

| Risk | Mitigation |
|---|---|
| Path-scoped `Write` permissions may not behave as specified | Verified in Phase 2; documented fallback in §6 |
| Frontend CI pins a backend commit that can go stale | `/contract` and the nightly drift check surface it; the pin is deliberately visible in review |
| Eleven agents is more than one person will use | Phase 4 exercises five. Unused definitions cost nothing at rest; they are only loaded when invoked |
| `gh` absent, so CI cannot be validated locally | Install `gh` during Phase 0, or validate on first push |
| Backend has uncommitted security-surface work | It is finding #1 of the first slice, before anything else is built on top of it |
