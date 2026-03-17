---
name: commit-push
description: Auto-generate a commit message by comparing current changes to the last commit, then push
disable-model-invocation: true
allowed-tools:
  - Bash
---

Follow these steps:

1. Run `git status` to see all changed and untracked files
2. Run `git diff HEAD` to compare all current changes against the last commit (both staged and unstaged)
3. Run `git log -3 --oneline` to understand recent commit style
4. Analyze the diff and auto-generate a concise commit message that:
   - Summarizes what changed and why
   - Follows the style of recent commits in the repo
   - Uses imperative mood (e.g. "Add", "Fix", "Update")
5. Stage all relevant changed files (avoid staging secrets like .env or large binaries)
6. Commit with the generated message, appending:
   ```
   Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
   ```
7. Push to the current branch's remote tracking branch
8. Report the commit message and push result
