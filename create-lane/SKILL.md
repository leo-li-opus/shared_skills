---
name: create-lane
description: Create a lane (swim lane) environment by triggering the release-lane GitHub Actions workflow
allowed-tools:
  - Bash
  - Read
  - Glob
  - AskUserQuestion
---

Create a lane environment for isolated testing/development. Follow these steps:

## 1. Auto-detect context

Run these in parallel:
- Get the current git branch name: `git branch --show-current`
- Get the current git user name: `git config user.name`
- Read the services config from `.github/workflows/config/services.yaml` in the repo root to get the list of deployable services

## 2. Build the service list

From `services.yaml`, filter to only services where `skip_deployment` is NOT true. These are the lane-deployable services. Do NOT hardcode this list — always read it dynamically from the file.

## 3. Ask the user for inputs

First, print the full numbered list of deployable services (from step 2) so the user can see all options. Format them in a clean numbered table.

Then use AskUserQuestion to ask **two questions** in a single call:

### Question 1: Services
Ask "Which services do you want to deploy? (enter numbers or names, e.g. '1,3,8' or 'api,hitl-worker')" with these preset options:
- "agent-web,api,growth-worker" (Labs)
- "agent-web,api,growth-worker,render" (Story Mode)
- "All services" (Deploy everything)

The user can select "Other" to type specific service numbers or names.

Parse the user's response — if they provide numbers, map them back to service names. If they provide names, use them directly.

### Question 2: Environment
Ask "Which environment?" with options:
- staging (Recommended)
- production

## 4. Generate lane name and TTL

- **Lane name**: Use the git username, lowercased, with spaces replaced by hyphens. Example: `allen-zhou`
- **Branch**: Use the current git branch (auto-detected, do NOT ask)
- **TTL**: Calculate as current Unix timestamp + 86400 (1 day from now). Use `date +%s` to get current timestamp, then add 86400.

## 5. Trigger the workflow

Run the `gh` command to trigger the lane workflow:

```bash
gh workflow run release-lane.yml \
  -f lane_name="<generated-lane-name>" \
  -f services="<comma-separated-services>" \
  -f environment="<selected-env>" \
  -f lane_ttl="<calculated-ttl>" \
  -f owner_name="<git-user-name>"
```

## 6. Confirm

Print a summary of what was triggered:
- Lane name
- Branch
- Services
- Environment
- TTL (as a human-readable date)
- Link: Tell the user to check GitHub Actions for progress
