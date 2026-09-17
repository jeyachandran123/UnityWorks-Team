---
name: ai-ml-architecture
description: Use when changing, debugging or evaluating anything in a repository that calls a model or depends on model output - LLM or VLM calls, chat profiles, thinking/reasoning budgets, prompts, provider or model configuration, embeddings, vector-store retrieval, RAG, grounding and citations, document extraction, generated-code sandboxes, agent pipelines, or evaluation datasets - and when a reply is blank, refused, ungrounded, slow, ignores a prompt edit, or retrieval returns poor results.
---

# AI/ML Architecture

## Overview

Applications that use models usually have **several independent model paths**, each with its own
client, provider switch and prompt. Most wasted effort comes from editing the wrong one.

> **Find the path that actually serves the request before touching a prompt, a model or a threshold.**

The second rule: every threshold, token budget and prompt rule should carry the measurement that
justified it. Change one only with a new measurement, and write the numbers next to it.

## Step 0 — Project facts first

Nothing below names a file in any particular project. Before starting, read what this repository says
about itself, in this order, and prefer it over anything generic here:

1. `.claude/team/roles/ai-ml-architect.md` — the project's brief for this role, if present.
2. Any project skill in `.claude/skills/` whose description covers models, prompts or retrieval —
   typically a map of the project's model paths, profiles, prompt locations and config traps.
3. `CLAUDE.md`, and `.claude/team.conf` (`gate` lines = how to run its tests).

If the project has no such map, build one with Steps 1–4 and offer it to the user as a project skill.

## Step 1 — Which path serves this request?

Discover, don't assume:

```bash
# model clients and provider switches
grep -rnE "OpenAI\(|AsyncOpenAI|anthropic|ollama|/chat/completions|generate\(|embeddings?\(" --include=*.py --include=*.ts --include=*.js . | grep -v node_modules | head -50
# where requests enter (routes, handlers, CLI, jobs)
grep -rnE "@(app|router)\.(get|post)|app\.(get|post)\(|export (async )?function (GET|POST)" . | grep -v node_modules | head -50
# settings that choose providers, models, feature flags
grep -rniE "(provider|model|enabled|flag)[a-z_]*\s*[:=]" --include=*config* --include=*settings* . | head -50
```

For the behaviour in question, trace **one concrete request** from its entry point: every early return,
feature flag and fallback in order, until the call that reaches a model. Record:

| Entry point | Conditions checked, in order | Handler that answered | Client + provider switch | Prompt source |

Consequences to state explicitly: which prompt files are *not* on this path, which flags default on,
and which settings exist but are read by nothing (grep each setting name for readers).

## Step 2 — The model-call layer

- **Call sites should name a profile or purpose, never a model name, temperature or token budget.**
  Look for a profiles/presets module; flag literals at call sites.
- **Reasoning/thinking output** must stay separate from the answer: not saved, not fed back into
  memory, streamed as its own event if shown.
- **Retries:** only for transient statuses (408/409/425/429/5xx); never 4xx. A stream may only be
  retried before its first event — after that a retry repeats text already shown.
- **Thinking budgets:** a model that spends its whole output budget thinking must raise, not return an
  empty "success". Fix by raising the budget or disabling thinking, never by swallowing the error.
- **Thinking switches differ by model family**, and the wrong key is usually silently ignored — verify
  the key against the provider's docs for the configured model.
- **Tests never reach a real provider:** use or build a fake client.

## Step 3 — Prompts

Locate the prompt for the path from Step 1 — hard-coded strings, template registries and versioned
prompt files often coexist, and only one is live. Lessons that transfer across models:

- **State the wanted behaviour; don't quote the unwanted one.** Quoting a bad output as "don't do this"
  tends to reproduce it.
- **Restate critical rules after long context** (citations, format, refusal) — rules only at the top
  are dropped once sources are appended.
- **Send history as real role turns,** not a transcript pasted into one message.
- Provider adapters should not compose or prepend prompts; prompts are versioned in one place.

## Step 4 — Retrieval and embeddings

Trace: chunking → embedding (model, provider, query vs passage mode) → store/collection → similarity
score formula → ranking/MMR → context budget. Traps:

- **Score scales differ between stores and formulas** (`1 - d`, `1 - d/2`, raw distance). Never reuse
  a threshold across two of them.
- **Vectors from different models/providers are not comparable.** Changing the embedding model means
  re-embedding everything stored — plan a reindex.
- Check whether the provider actually honours query/passage modes and batching; many local ones don't.
- Hosted vector stores may not return embeddings, silently disabling MMR-style diversity.
- Token counts are often estimated (`len // 4`); budgets are approximate.

## Step 5 — Invariants to preserve

- **Grounding:** answers must cite sources that resolve and pass the grounding threshold. A deliberate
  refusal is a valid, ungrounded result — not an error. **Never "fix" refusals** by lowering the
  threshold, accepting uncited text or catching the validator's rejection; find out why retrieval or
  citation failed.
- **Provider neutrality:** no provider names branching inside core pipelines; providers register
  adapters.
- **Sandboxing:** generated code runs without network access; user data does not leave the machine.
- **Escalation:** high-stakes turns are escalated, never auto-answered; failures fall back safely.

Find the tests that hold each of these (`unityworks-team:invariant-audit` proves they fail when broken).

## Verification

- Run the narrowest tests for the path, using the project's own commands (`team.conf` `gate` lines or
  `CLAUDE.md`), then its full gate.
- **A prompt or threshold change needs real runs**, several of them, against the real path — one sample
  proves nothing. Record observed rates next to the change. Real runs spend model budget: do them only
  with the user's go-ahead; otherwise report the claim as **unmeasured** with the exact command and
  sample count that would settle it.

## Common mistakes

| Mistake | Instead |
|---|---|
| Editing a prompt that is not on the serving path | Trace the request first (Step 1) |
| Hard-coding a model name, temperature or `max_tokens` at a call site | Name a profile |
| Flipping a setting nothing reads | Grep for its readers first |
| Lowering a grounding threshold because answers get refused | Debug retrieval, citations, attribution |
| Switching embedding model without reindexing | Re-embed; old and new vectors can't be compared |
| Catching the error behind a blank reply | Raise the budget or turn thinking off |
| Quoting bad output in a prompt | Describe the required behaviour positively |
| Mixing reasoning text into the saved answer or memory | Keep reasoning separate |
| Changing a threshold on intuition | Measure on real data; write the numbers down |
