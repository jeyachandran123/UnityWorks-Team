---
description: Ask one specialist a single question — no full review, no report file
argument-hint: <role> <question>
allowed-tools: Bash, Agent
---

Arguments: $ARGUMENTS

1. Run `bash "${CLAUDE_PLUGIN_ROOT}/scripts/team-context"`; if it fails, relay the message and stop.
2. The first word is the role, the rest is the question. Roles: the plugin roles
   (`security-engineer`, `architecture-auditor`, `backend-engineer`, `frontend-engineer`,
   `ux-architect`, `devops-engineer`, `qa-engineer`, `product-analyst`, `release-scribe`, `ai-ml-architect` → agent type
   `unityworks-team:<role>`) and each `agent = <name>` in `$REPO_SELF/.claude/team.conf` (agent type
   `<name>`). If the role or question is missing, say what is needed and stop.
3. Dispatch that one agent with this prompt, filling in the paths and the question:

> This is a brief, not a review. Repository: <REPO_SELF> (related: <REPO_* values, or none>).
> **Do not write a report file.** Investigate only as far as the question needs. Answer in at most 250
> words: the answer first, then the evidence — each claim with a `file:line` or a command you ran and
> its output. If the question cannot be answered with evidence, say what would settle it.
>
> Question: <question>

Relay the agent's answer as it came back. Do not add your own conclusions to it.
