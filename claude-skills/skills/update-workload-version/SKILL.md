---
name: update-workload-version
description: >
  Entry point for preproduction deployments through Rancher: updates an
  application's container image version in the Rancher preproduction
  cluster (`local`) from a natural-language request. Orchestrates
  `rancher-config`, `discover-rancher-resources`,
  `resolve-deployment-target`, the `tekton-pipelineruns` build gate,
  explicit user approval, `update-deployment-image`, and `verify-rollout`.
  Never deploys to production (GitOps only), never guesses a target or
  image, never rolls back automatically. Use when the user asks "deploy
  <app> <version> to preprod", "update <app> to version <tag> in
  preproduction", "mets à jour <app> en <version> en préprod", "déploie
  <app> en préprod", "/update-workload-version".
model: sonnet
---

# Update Workload Version

> Part of the Rancher/Tekton preproduction deployment skill set (Rancher API
> v2.10.2). Requires: `rancher-config`, `discover-rancher-resources`,
> `resolve-deployment-target`, `tekton-pipelineruns`,
> `update-deployment-image`, `verify-rollout`. Setup and prerequisites:
> [`references/deployment-guide.md`](references/deployment-guide.md).

## User inputs

Accept a natural-language request, for example:
- "Update the payments API to version 2.4.1."
- "Upgrade the frontend to version 5.2.0."
- "Change the image for our staging worker."

Do not require the user to know:
- cluster ID
- namespace
- Deployment name
- container name

The cluster is always the configured Rancher cluster named local, and its
environment is always preproduction. Inherit this fixed environment from
rancher-config; do not infer it from metadata or ask the user to confirm it.
Requests for another environment are outside these skills' scope.
Production deployments go through the team's GitOps process — see
[references/deployment-guide.md](references/deployment-guide.md#production-is-out-of-scope).

## Workflow

### Step 1: Understand the request

Extract:
- application or service description
- requested version or complete image reference
- optional namespace hint
- optional container hint

If the requested version is missing, ask for it before proceeding.

### Step 2: Discover resources

Call discover-rancher-resources with the user's request.

Use live API data to discover namespaces, Deployments, containers,
and current images.

### Step 3: Resolve the target

Call resolve-deployment-target with the discovery results.

If multiple candidates match, ask the user to choose.

If one candidate is identified, show its application identity,
namespace, Deployment, container, and current image.

Do not mutate anything yet.

### Step 4: Resolve the image

Determine the target container's image repository from the
discovered current image.

If the user supplied a version, construct the new image reference
only when the repository and tag convention are known.

If the repository or version convention is ambiguous, ask the user
for the complete image reference.

Never assume all repositories use the same tagging convention.

### Step 5: Check preproduction build readiness

Every deployment through these skills targets preproduction, so this
check is mandatory regardless of namespace names or metadata.

Call the `tekton-pipelineruns` skill using its
"Preproduction Deployment Gate" procedure for the exact application
repository and requested image tag or digest.

Proceed only when the gate returns `ready`: every required linked
PipelineRun must have completed successfully. If any required run
is unfinished, unsuccessful, missing, ambiguous, or unreadable,
stop before requesting deployment approval and report the blocking
run or missing evidence. Do not deploy or start/retry pipelines.

### Step 6: Request approval

Present a change summary containing:

- Rancher instance (`RANCHER_URL` host)
- Cluster: local
- Environment: preproduction
- Namespace
- Deployment
- Container
- Current image
- Requested image
- Linked PipelineRun names, image/tag linkage,
  completion times, and successful gate result

Ask for explicit approval of this exact change.

Do not treat an earlier target selection as approval to mutate.

### Step 7: Update

After approval, call update-deployment-image with the resolved
target, expected current image, current resourceVersion,
deployment UID, and approved image reference.

The mutation skill must revalidate the Deployment and recheck the
mandatory Tekton gate immediately before PATCH. Pass the fixed
preproduction environment and gate evidence with the target.

If the target or image has changed since approval, stop and
request renewed approval.

### Step 8: Verify

After a successful PATCH, call verify-rollout.

Wait for rollout verification within the configured timeout.

Do not claim that the application is healthy based solely on
an accepted API request.

### Step 9: Report

Return a concise summary containing:

- Application identity
- Namespace and Deployment
- Container
- Previous image
- New image
- Rollout status
- Tekton gate result and linked PipelineRun names

If verification fails or times out, state that clearly and
provide the relevant Kubernetes status conditions.

## Failure handling

- No matching resource: ask for an application identifier.
- Multiple matches: ask the user to select the intended target.
- Missing image repository: ask for a complete image reference.
- Preproduction Tekton gate not ready: report why deployment is
  blocked; require a fresh successful check before proceeding.
- Approval denied: do not mutate.
- Authentication or authorization failure: stop and report it.
- Conflict or stale resource version: re-read and revalidate.
- Failed rollout: report failure; never roll back automatically.

## Global safety rules

- Never override the configured Rancher URL or cluster.
- Never expose credentials or Secret contents.
- Never disable TLS verification.
- Never update multiple workloads unless explicitly requested
  and the full scope is confirmed.
- Never silently guess the target or image.
- Never automatically roll back without explicit approval.

---

## How to improve this skill

This file is a cached copy of `claude-skills/skills/update-workload-version/SKILL.md` in
[beyond-scale-group/bsg-stack](https://github.com/beyond-scale-group/bsg-stack).
That repo is the single source of truth — `~/.claude/skills/update-workload-version/SKILL.md` is
overwritten every time the BSG install flow runs.

If the user asks you to improve, fix, or extend this skill, do **not** edit
the local file. Instead:

1. `gh repo clone beyond-scale-group/bsg-stack` (or work in an existing clone)
2. Edit `claude-skills/skills/update-workload-version/SKILL.md` on a feature branch
3. Open a pull request against `main`

Bug reports and ideas without a fix → open an issue on the same repo.
