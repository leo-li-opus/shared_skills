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

## 6. Confirm and print summary

Print a summary of what was triggered:
- Lane name
- Branch
- Services
- Environment
- TTL (as a human-readable date)

## 7. Monitor the workflow

After triggering, wait 5 seconds then find the run ID:

```bash
gh run list --workflow=release-lane.yml --limit=1 --json databaseId,status,conclusion --jq '.[0]'
```

Poll the workflow status every 30 seconds using `gh run view <run-id>`. The single workflow run contains multiple parallel jobs (one per service in the matrix). While polling:

- Print a brief status update each time showing per-job status, e.g.:
  ```
  Still running... (2m elapsed)
    Release api        ✓ completed
    Release hitl-worker  ⏳ in_progress
    Release render       ⏳ queued
  ```
  Use `gh run view <run-id> --json jobs --jq '.jobs[] | {name, status, conclusion}'` to get per-job status.

- If any job fails, immediately fetch its logs:
  ```bash
  gh run view <run-id> --log-failed
  ```
  Print the failed job name and the relevant error excerpt.

- Continue monitoring remaining jobs even if one fails (the workflow uses `fail-fast: false`).

- When all jobs complete, print a final summary:
  - **Per-service result**: which services succeeded and which failed
  - For failures: show the error from GH Actions logs, then run kubectl diagnostics (see step 8)
  - For success: confirm the lane is ready and print access links (see step 9)

## 8. Diagnose failures with kubectl

If any service deployment failed, use kubectl to inspect the cluster state for that service. The lane deployments go to:
- **Staging cluster**: `opus-ai-staging-opusagent-flagship`
- **Production cluster**: `opus-agent-prod-flagship`

Lane pods use the naming pattern `<service-name>-<lane-name>` (e.g., `api-allen-zhou`, `engine-worker-allen-zhou`). The `fullnameOverride` in each service's `values.yaml` defines the base service name.

Run these kubectl commands for each failed service:

```bash
# 1. Find the lane pods
kubectl get pods -l "app.kubernetes.io/instance=<service-name>-<lane-name>" --context <cluster> -A

# 2. If no pods found, check the deployment/release
kubectl get deployments -l "app.kubernetes.io/instance=<service-name>-<lane-name>" --context <cluster> -A

# 3. If pods exist but are failing, check pod status and events
kubectl describe pod -l "app.kubernetes.io/instance=<service-name>-<lane-name>" --context <cluster> -A | tail -50

# 4. Check container logs for crash details
kubectl logs -l "app.kubernetes.io/instance=<service-name>-<lane-name>" --context <cluster> -A --tail=100
```

Summarize the findings:
- **CrashLoopBackOff**: Show the container logs with the crash reason
- **ImagePullBackOff**: The image may not have been built — suggest checking the build step
- **Pending**: Check events for scheduling issues (resource limits, node availability)
- **OOMKilled**: Pod ran out of memory — suggest increasing memory limits
- If kubectl is not configured or access is denied, tell the user and suggest they check manually

## 9. Print access links

After successful deployment, print the appropriate links based on which services were deployed:

### If agent-web was deployed (web change)
The lane web app is deployed to Cloudflare Pages. Get the deployment URL from the workflow logs — it follows the pattern:
- Staging: `https://<commit-hash>.opus-agent-web-staging.pages.dev`
- Production: `https://<commit-hash>.opus-agent-web-prod.pages.dev`

This lane web deployment already routes API requests to the lane backend automatically via the `X-OPUS-LANE-NAME` header.

Print the Cloudflare Pages URL as the primary access link.

### If agent-web was NOT deployed (backend-only change)
The user should use the standard web app with a query parameter to route to their lane backend:
- Staging: `https://opus-agent-web-staging.pages.dev/?debugLaneName=<lane-name>`
- Production: `https://opus-agent-web-prod.pages.dev/?debugLaneName=<lane-name>`

The `debugLaneName` query param sets the `X-OPUS-LANE-NAME` header on all API requests, routing them to the lane backend workers.

### Always print
- The lane name for manual use in the Debug Panel (Projects → Debug Panel → Origin Changer → enter lane name)
- The GitHub Actions run URL for reference
