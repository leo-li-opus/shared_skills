---
name: create-pr
description: Create a pull request for the current branch
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
  - AskUserQuestion
---

Create a pull request for the current branch. Follow these steps carefully:

## Pre-flight checks

Before creating the PR, look for CI configuration to understand what checks are required:

1. Check for a `Makefile` — look for targets like `check`, `lint`, `test`, `fmt`, `ci`
2. Check for `.github/workflows/` — read PR-triggered workflows to understand CI requirements
3. Check for lint-staged, pre-commit hooks (`.husky/`, `.pre-commit-config.yaml`)
4. Check for a `CLAUDE.md` or `AGENTS.md` that documents dev commands

Run the appropriate local checks (lint, format, test, type-check) based on what you find. Run them in parallel where possible. If any check fails, attempt to auto-fix (e.g. formatter, lint --fix) before asking the user.

## PR title format

PR titles MUST pass this CI validation regex:

```
/^(\[[^\]]+\] )?(feat|hotfix|fix|docs|style|refactor|perf|test|chore|build|ci|revert): [a-z](?!.*  )(?!.*--).{7,68}[^. ]$/
```

Rules:
- Lowercase type prefix (feat, fix, chore, etc.)
- Single space after colon
- Lowercase first letter of description
- 8-70 characters total length
- No ending period
- No double spaces
- No double dashes
- Optional scope prefix in brackets: `[web] fix: description`
- Ticket IDs (e.g. AO-1234, LABS-29) MUST be included at the end of the PR title (e.g. `feat: add zod module LABS-29`). Also include them in the body with a magic word.

## PR requirements

Check `.github/workflows/` for additional PR validation rules. Common checks include:

- **PR body** — must not contain template placeholder text; write an actual summary
- **Ticket linking** — include the ticket ID in the PR body
- **No secrets** — ensure no credentials are committed
- **File size limits** — some repos enforce max file sizes

## Gather missing information

Before creating the PR, check if you have enough info. Use AskUserQuestion to ask for anything unclear:

- **Linear ticket ID**: ALWAYS ask the user for a Linear ticket number (e.g. `AO-1234`) using AskUserQuestion, even if one appears in the branch name — confirm it.
- **PR summary**: If the changes are ambiguous or could be described multiple ways, ask the user for a brief description of the intent.
- **Target branch**: Default to `main` or the repo's default branch. If the branch name suggests otherwise, confirm with the user.

## Creating the PR

1. Check git status, diff against base branch, and recent commits
2. Push the branch if needed
3. Create the PR as a **draft** with `gh pr create --draft` using a clear title and structured body with `## Summary` (what changed and why) and `## How to test` (brief description of how to verify, NOT a long checkbox list — keep it to 1-3 sentences)
4. Return the PR URL when done

If the user provides additional context, incorporate it into the PR description.
