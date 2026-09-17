---
name: evidence-report
description: Use when writing a review, audit, specialist finding or triage report for the UnityWorks Vision AI repositories, including anything filed under docs/reviews/ — and whenever a unityworks-team agent produces its output.
---

# Evidence Report

## Overview

Reports from independent specialists are only comparable, deduplicable and rankable if they share
one shape. This is that shape. A finding without a location and executed evidence is an opinion and
is not filed.

## Where it goes

`docs/reviews/<YYYY-MM-DD>/<role>.md` inside the repository reviewed. Date is today (`date +%F`).
A review spanning both repos files one report per repo. `tech-lead` writes `00-triage.md` in the same
directory.

## The file, exactly

```markdown
---
role: security-engineer
repo: unityworks-vision-ai-backend
ref: <output of `git rev-parse --short HEAD`, plus " + working tree" if `git status --short` is non-empty>
date: YYYY-MM-DD
scope: <what was examined, e.g. "app/auth/**, app/authorization/**">
---

# Summary

<2–4 sentences: the verdict first, then the count by severity.>

## Finding 1 — <one-sentence claim, stated as a fact that is either true or false>
- **Severity:** blocker | major | minor | note
- **Location:** app/auth/tokens.py:88
- **Evidence:** <the command you ran, then its real output, trimmed to the relevant lines>
- **Failure scenario:** <concrete input or state → the wrong behaviour it produces>
- **Recommendation:** <the change, and the test that would prove it>

## Verified clean
- <area> — <what was checked, and the command that showed it>
```

## Severity, defined

| Severity | Means |
|---|---|
| **blocker** | Must not ship: data exposure, cross-tenant access, broken invariant, gate red, data loss |
| **major** | Wrong behaviour a user or operator will hit; a guard test that is a tautology |
| **minor** | Real defect with a narrow or unlikely path |
| **note** | Drift, docs, clarity — no behaviour change |

## Rules

- Every finding has a `file:line` that exists at `ref`.
- **Evidence** is something you executed — a test run, a grep, a curl, a mutation — with its output.
  "Reading the code shows…" is a location, not evidence; pair it with a command that demonstrates it.
- **Failure scenario** names inputs. "Could be insecure" is not a scenario.
- Areas examined and found correct go under **Verified clean**, so silence is never ambiguous.
- One claim per finding. Two problems in one function are two findings.
- Do not propose edits as diffs to apply; recommend. Implementation happens in the main session.
- If a check could not run (missing dependency, no network), say so under the finding or in Summary.
  Never report an unrun check as passing.

## Triage file (`00-triage.md`, tech-lead only)

```markdown
---
role: tech-lead
date: YYYY-MM-DD
inputs: [security-engineer.md, qa-engineer.md, …]
---

# Verdict
<ship | ship after blockers | do not ship> — one paragraph.

## Remediation order
| # | Finding (source#n) | Severity | Depends on | Why this position |

## Merged duplicates
- security-engineer#2 = qa-engineer#4 — <kept wording from which, and why>

## Contradictions
- <A says X, B says not-X> — <which evidence is stronger, or what to run to settle it>
```
