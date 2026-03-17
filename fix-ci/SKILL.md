# Fix CI

Watch GitHub Actions for the current PR, pull back failure logs, fix the code, push, and repeat until all checks pass.

## Steps

### 1. Find the current PR

```bash
gh pr view --json number,url,headRefName
```

If no PR exists, tell the user and stop.

### 2. Wait for GitHub Actions to complete

Poll until all runs triggered by the latest push reach a terminal state:

```bash
# Get the latest commit SHA on this branch
COMMIT_SHA=$(git rev-parse HEAD)

# List runs for this commit
gh run list --commit $COMMIT_SHA --json databaseId,name,status,conclusion
```

Poll every 20 seconds. Print a brief status line each poll:
```
⏳ CI running... (1m 20s) [code-quality: in_progress] [check-pr-basics: success]
```

Wait up to 20 minutes. If CI hasn't finished by then, tell the user and stop.

### 3. Check results

```bash
gh pr checks
```

If all checks pass → print "✅ All CI checks passed!" and stop.

If checks are failing → proceed to step 4.

### 4. Pull failure logs from GitHub Actions

For each failing run, fetch only the failed step logs:

```bash
gh run view <run-id> --log-failed
```

Format the output clearly:
```
❌ FAILING CHECKS:

--- [check-pr-basics / Check Ticket Number] ---
<relevant log lines>

--- [code-quality / typescript] ---
<relevant log lines>
```

Truncate each section to 150 lines if needed (note if truncated).

### 5. Fix the code

Analyze the logs and edit the source files to fix each issue. Common cases:

- **TypeScript / lint errors** → edit the relevant source files
- **Test failures** → fix the code under test or update the test if the behavior changed intentionally
- **PR title/body/ticket format** → `gh pr edit --title "..." --body "..."`
- **File size violation** → identify the offending file in the log and fix it
- **Forbidden domain / secrets scan** → investigate carefully, alert the user before changing anything

Do NOT run any local checks or build commands — just edit the files.

### 6. Commit and push

```bash
git add <changed files>
git commit -m "fix: address CI failures"
git push
```

Then go back to step 2 and wait for the new CI run triggered by this push.

### 7. Repeat

Keep iterating until all checks pass or 3 fix attempts have been made. After 3 attempts, summarize what was tried and what is still failing, then ask the user for guidance.

## Notes

- Never force-push or use `--no-verify`
- If a failure is ambiguous or touches security/secrets, pause and ask the user
- If `$ARGUMENTS` are provided, treat them as hints about what's failing
