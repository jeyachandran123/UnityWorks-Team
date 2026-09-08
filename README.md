# unityworks-team

The Claude Code plugin for the UnityWorks Vision AI repositories.

Holds what is identical across `unityworks-vision-ai-backend` and
`unityworks-vision-ai-frontend`: specialist role definitions, cross-repo
procedures, shared hook scripts, and the slash commands that drive them.

Repo-specific procedures live in each repository's own `.claude/skills/`,
so they are reviewed in the same pull request as the code they describe.

Design: `../docs/superpowers/specs/2026-09-08-unityworks-agent-team-design.md`

## Install

    claude plugin marketplace add ./
    claude plugin install unityworks-team@unityworks-local

## Layout

    .claude-plugin/   manifest + local marketplace
    hooks/            shared hook scripts (extensionless) + run-hook.cmd
    tests/            hook tests — run with: bash tests/run-tests
