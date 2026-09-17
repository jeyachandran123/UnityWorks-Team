---
description: Ask one UnityWorks specialist a single question — no full review, no report file
argument-hint: <role> <question>
allowed-tools: Agent
---

Arguments: $ARGUMENTS

The first word is the role (`security-engineer`, `architecture-auditor`, `vision-specialist`,
`backend-engineer`, `frontend-engineer`, `ux-architect`, `devops-engineer`, `qa-engineer`,
`product-analyst`, `release-scribe`, `ai-ml-architect`); the rest is the question. If either is missing, say what is
needed and stop.

First locate the repositories by their contents, never by folder name. If the command exits
non-zero, relay its message and stop.

- Role `ai-ml-architect`: `bash "${CLAUDE_PLUGIN_ROOT}/scripts/find-repo" ai-assistant-backend`
- Any other role: `bash "${CLAUDE_PLUGIN_ROOT}/scripts/locate-pair" --print`

Then dispatch one `unityworks-team:<role>` agent with this prompt, filling in the question and
replacing `<REPOS>` with the absolute path(s) just printed:

> This is a brief, not a review. Answer one question for <REPOS>. **Do not write a report file.** Investigate only as far as the question needs. Answer in
> at most 250 words: the answer first, then the evidence — each claim with a `file:line` or a command
> you ran and its output. If the question cannot be answered with evidence, say what would settle it.
>
> Question: <question>

Relay the agent's answer as it came back. Do not add your own conclusions to it.
