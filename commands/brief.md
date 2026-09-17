---
description: Ask one UnityWorks specialist a single question — no full review, no report file
argument-hint: <role> <question>
allowed-tools: Agent
---

Arguments: $ARGUMENTS

The first word is the role (`security-engineer`, `architecture-auditor`, `vision-specialist`,
`backend-engineer`, `frontend-engineer`, `ux-architect`, `devops-engineer`, `qa-engineer`,
`product-analyst`, `release-scribe`); the rest is the question. If either is missing, say what is
needed and stop.

Dispatch one `unityworks-team:<role>` agent with this prompt, filling in the question:

> This is a brief, not a review. Answer one question for the UnityWorks Vision AI repositories
> (`unityworks-vision-ai-backend`, `unityworks-vision-ai-frontend`, found beside or above the current
> directory). **Do not write a report file.** Investigate only as far as the question needs. Answer in
> at most 250 words: the answer first, then the evidence — each claim with a `file:line` or a command
> you ran and its output. If the question cannot be answered with evidence, say what would settle it.
>
> Question: <question>

Relay the agent's answer as it came back. Do not add your own conclusions to it.
