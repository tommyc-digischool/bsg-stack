---
name: tekton-pipelineruns
description: >
  Inspect, monitor, and diagnose Tekton PipelineRuns and TaskRuns
  exclusively in the `tekton-pipelines` namespace using authenticated
  Rancher HTTP APIs. Reuses the `rancher-config` skill for credentials and
  cluster resolution. Strictly read-only: status tables, failed-build root
  cause with log excerpts, and the "Preproduction Deployment Gate" that
  `update-workload-version` / `update-deployment-image` require before any
  image update. Never uses kubectl or a browser; never restarts, cancels,
  or retries runs. Use when the user asks "show the latest PipelineRuns",
  "are my Tekton pipelines green?", "why did the last build fail?",
  "show the error logs from the last build", "le build Tekton est passé ?",
  "pourquoi la pipeline a échoué ?".
model: sonnet
---

# Tekton PipelineRuns

> Part of the Rancher/Tekton preproduction deployment skill set
> (Rancher API v2.10.2). Requires: `rancher-config`. Setup and
> prerequisites:
> [`update-workload-version/references/deployment-guide.md`](../update-workload-version/references/deployment-guide.md).

## Purpose

Inspect Tekton PipelineRuns, TaskRuns, and execution logs through
the Rancher Kubernetes HTTP API.

This skill is strictly read-only.

Never use kubectl, direct Kubernetes connections, browser automation,
or the Tekton Dashboard.

## Dependencies

Require the existing `rancher-config` skill.

Before making any Tekton request:

1. Load and follow the `rancher-config` skill.
2. Authenticate using its configured Rancher credentials.
3. Resolve the Rancher cluster named `local`.
4. Obtain the resolved `kubernetes_api_base_url`.
5. Reuse its HTTP Basic authentication configuration.

Do not duplicate Rancher configuration or credentials.

If `rancher-config` is unavailable, authentication fails, or cluster
resolution fails, stop and explain the problem.

## Fixed Configuration

The Kubernetes namespace is always:

`tekton-pipelines`

Use this namespace for all PipelineRun, TaskRun, Pod, and Pod log
requests.

Rules:

- Never ask the user to specify a namespace.
- Never accept a namespace override.
- Never discover or enumerate other namespaces.
- Never issue cluster-wide resource listing requests.
- Never use the namespace `default`.
- Never override the Rancher host configured by `rancher-config`.

The Kubernetes proxy base URL must come from `rancher-config`.

Do not hardcode the resolved cluster ID.

## Authentication

Follow the authentication instructions in `rancher-config`.

Use HTTP Basic authentication:

- Username: Rancher Access Key
- Password: Rancher Secret Key

Generate the Authorization header using the HTTP client.

Never use the Secret Key alone as a Bearer token.

Security requirements:

- Never print credentials.
- Never expose Authorization headers.
- Never store credentials in skill files.
- Never include credentials in URLs.
- Never disable TLS certificate verification.
- Never redirect authenticated requests to another host.

## API Configuration

Use the Kubernetes HTTP API exposed through Rancher.

Base URL:

`{kubernetes_api_base_url}`

### Tekton API Discovery

Before requesting Tekton resources, discover supported API versions:

`GET {base}/apis/tekton.dev`

Prefer `v1` when available.

If `v1` is unavailable, select a compatible served version.

If no supported Tekton API version is available, stop and report
the problem.

Cache the discovered API version for the current workflow.

Do not assume that all installations expose identical API versions.

## HTTP Request Rules

All requests must:

- Use HTTPS.
- Use HTTP GET only.
- Reuse the Rancher authentication configuration.
- Enforce TLS verification.
- Remain on the configured Rancher host.
- Use URL-encoded path components and query parameters.
- Set reasonable connection and response timeouts.
- Avoid printing sensitive response data unnecessarily.

For Kubernetes list responses, follow pagination using
`metadata.continue` until the requested result set is complete.

Do not silently report partial results as complete.

## Available Operations

### 1. List PipelineRuns

Endpoint:

`GET {base}/apis/tekton.dev/{version}/namespaces/tekton-pipelines/pipelineruns`

Retrieve PipelineRuns from the fixed namespace.

Support:

- Listing recent PipelineRuns.
- Filtering by PipelineRun name.
- Filtering by status.
- Returning the latest N PipelineRuns.
- Sorting by creation timestamp.
- Handling paginated API responses.

Extract:

- PipelineRun name.
- Namespace.
- Creation timestamp.
- Pipeline reference.
- Start time.
- Completion time.
- Status.
- Reason.
- Message.

Sort by creation timestamp descending unless the user requests
another ordering.

#### Status Interpretation

Inspect:

`status.conditions`

Find the condition where:

`type == "Succeeded"`

Interpret its `status` value:

| Value | Interpretation |
|-------|----------------|
| True | Succeeded |
| False | Failed or otherwise terminated unsuccessfully |
| Unknown | Running, pending, or unresolved |
| Missing | Unknown |

Use the condition's `reason` and `message` to distinguish:

- Pending.
- Running.
- Succeeded.
- Failed.
- Cancelled.
- Timed out.

Do not automatically classify every `Unknown` condition as Running.

Do not infer success solely from a completion timestamp.

### 2. Inspect a PipelineRun

Endpoint:

`GET {base}/apis/tekton.dev/{version}/namespaces/tekton-pipelines/pipelineruns/{name}`

Retrieve the requested PipelineRun.

Extract:

- Name.
- Creation timestamp.
- Pipeline reference.
- Overall status.
- Start time.
- Completion time.
- Failure reason.
- Failure message.
- Child references.
- Relevant task execution details.

If the PipelineRun succeeded, provide a concise summary.

If the PipelineRun failed, investigate its TaskRuns.

If the PipelineRun is still running, report its current progress
without inventing a completion time.

### 3. Find Related TaskRuns

First inspect the PipelineRun field:

`status.childReferences`

Select references where:

`kind == "TaskRun"`

If child references are unavailable or incomplete, list TaskRuns
using the Kubernetes label selector:

`tekton.dev/pipelineRun={pipelinerun-name}`

Endpoint:

`GET {base}/apis/tekton.dev/{version}/namespaces/tekton-pipelines/taskruns`

Pass the label selector as a URL-encoded query parameter.

For each related TaskRun, retrieve:

- Name.
- Status.
- Reason.
- Message.
- Start time.
- Completion time.
- Associated Pod name.
- Step execution statuses.

Do not assume all PipelineRuns expose `childReferences`.

Do not treat an empty TaskRun result as proof that no task executed.

If TaskRuns cannot be discovered, report the limitation.

### 4. Inspect a TaskRun

Endpoint:

`GET {base}/apis/tekton.dev/{version}/namespaces/tekton-pipelines/taskruns/{name}`

Inspect:

- `metadata.name`
- `status.conditions`
- `status.steps`
- `status.podName`
- `status.startTime`
- `status.completionTime`

Find the `Succeeded` condition.

Identify failed steps using the available step statuses.

For each failed step, report:

- Step name.
- Termination reason.
- Exit code, if present.
- Termination message, if present.
- Start and completion timestamps, if available.

Do not invent missing step details.

### 5. Retrieve Execution Logs

Retrieve Pod logs only when needed to investigate a failure
or when explicitly requested.

#### Step 1: Identify the Pod

Read:

`status.podName`

from the TaskRun.

If no Pod name is available, report that logs cannot currently
be retrieved through the Pod API.

#### Step 2: Retrieve Logs

Endpoint:

`GET {base}/api/v1/namespaces/tekton-pipelines/pods/{podName}/log`

Pass the container name using the query parameter:

`container={containerName}`

Prefer retrieving a bounded amount of logs using supported
Kubernetes log query parameters such as `tailLines`.

If the container restarted and earlier logs are relevant,
request previous logs using:

`previous=true`

Do not retrieve previous logs unless necessary.

#### Log Handling

- Prefer relevant excerpts rather than full logs.
- Identify the step and container associated with the error.
- Include useful surrounding context.
- Redact credentials, tokens, and other secrets.
- Do not expose sensitive environment variables.
- Do not execute commands inside Pods.
- Do not retry indefinitely when logs are unavailable.

## Failure Analysis Workflow

When the user asks why a PipelineRun failed:

1. Retrieve the PipelineRun.
2. Inspect its `Succeeded` condition.
3. Extract the failure reason and message.
4. Discover related TaskRuns.
5. Identify TaskRuns that failed.
6. Inspect failed step statuses.
7. Identify associated Pods.
8. Retrieve relevant container logs when available.
9. Determine the most likely root cause.
10. Suggest concrete corrective actions.

Separate confirmed facts from hypotheses.

Never claim a root cause is confirmed without supporting evidence.

If the failure information is insufficient, explain what is missing.

## Preproduction Deployment Gate

Read-only check consumed by `update-workload-version` (before asking for
approval) and `update-deployment-image` (again, immediately before
PATCH). It answers one question: have all builds of this exact artifact
finished successfully?

### Inputs

- application: application or source repository identity
- image_reference: complete image reference being deployed
  (repository plus tag or digest)

### Procedure

1. List PipelineRuns in `tekton-pipelines` (operation 1), following
   pagination. Always fetch fresh data; never reuse an earlier result.
2. Select the PipelineRuns linked to this exact artifact. A run is
   linked only on explicit evidence, for example:
   - a `spec.params` value or a PipelineRun/TaskRun result (such as
     `IMAGE_URL`, `IMAGE_DIGEST`, an image, tag, or revision parameter)
     equal to the image reference, its tag, or its digest;
   - labels or annotations naming the application repository together
     with the same tag, digest, or source revision.
   Do not link a run on name similarity alone.
3. Include every linked run, including runs created after an earlier
   check for the same artifact (re-runs, retriggers).
4. Evaluate each linked run's `Succeeded` condition (see Status
   Interpretation).
5. Return exactly one gate result:

| Result | Condition |
|--------|-----------|
| `ready` | At least one linked run, and every linked run is Succeeded |
| `blocked` | A linked run is pending, running, failed, cancelled, or timed out |
| `missing` | No PipelineRun can be linked to the artifact |
| `ambiguous` | Linkage evidence is weak or conflicting |
| `unreadable` | Runs could not be read (auth, RBAC, API, pagination incomplete) |

Only `ready` allows a deployment. Any other result blocks it; report
the blocking runs or the missing evidence. Never start, retry, or
cancel a PipelineRun to make the gate pass.

### Output

- gate_result
- application and image_reference checked
- linked PipelineRuns: name, status, completion time, linkage evidence
- blocking runs or missing evidence, when not `ready`
- checked_at timestamp

## Output Format

### PipelineRun Status Overview

Present a table:

| PipelineRun | Status | Started | Duration |
|-------------|--------|---------|----------|

Show the latest PipelineRuns first.

For running PipelineRuns, calculate elapsed time from their
start timestamp if available.

For completed PipelineRuns, calculate duration using their
start and completion timestamps.

Do not invent missing timestamps or durations.

### Failure Investigation

Use the following structure:

**PipelineRun:** PipelineRun name

**Status:** Failed

**Reason:** Failure reason

**Failed TaskRun:** TaskRun name

**Failed step:** Step name

**Exit code:** Exit code, when available

**Relevant logs:**

Include concise log excerpts supporting the diagnosis.

**Root cause:**

Explain the observed failure and supporting evidence.

**Suggested fix:**

Provide concrete troubleshooting or remediation steps.

Clearly distinguish confirmed causes from likely causes.

### Running PipelineRun

For active executions, report:

- PipelineRun name.
- Current status.
- Start time.
- Elapsed time.
- Completed TaskRuns.
- Running TaskRuns.
- Failed TaskRuns, if any.

Never predict a completion time without sufficient evidence.

## Error Handling

Handle HTTP errors explicitly.

### 401 Unauthorized

Report that Rancher authentication failed.

Do not print credentials.

Do not repeatedly retry with the same invalid credentials.

### 403 Forbidden

Report insufficient Rancher or Kubernetes RBAC permissions.

Identify the requested resource and operation.

Do not attempt to bypass authorization.

### 404 Not Found

Check whether:

- The resource exists.
- The requested API version is served.
- The resource path is correct.
- The cluster resolution succeeded.

Do not automatically assume a missing PipelineRun means
the user supplied an incorrect name.

### 429 Too Many Requests

Respect the `Retry-After` header where available.

Use limited retries.

### 5xx Server Errors

Report Rancher or downstream Kubernetes API errors.

For transient failures, use limited exponential backoff.

Never retry indefinitely.

### Network Errors

Report connection failures, TLS failures, and timeouts
without exposing credentials or sensitive headers.

Never disable TLS verification as a workaround.

## Required Kubernetes RBAC

The Rancher identity must have the following permissions
in the `tekton-pipelines` namespace.

| API Group | Resource | Verbs |
|-----------|----------|-------|
| tekton.dev | pipelineruns | get, list |
| tekton.dev | taskruns | get, list |
| core | pods | get |
| core | pods/log | get |

Do not require permissions to create, update, patch,
delete, or execute Kubernetes resources.

If listing Pods becomes necessary for a supported
diagnostic workflow, explicitly request the additional
`list` permission rather than assuming it exists.

## Safety Rules

- Operate only in the `tekton-pipelines` namespace.
- Never accept namespace overrides.
- Never issue cluster-wide resource listing requests.
- Never use kubectl.
- Never use a browser.
- Never access the Tekton Dashboard.
- Never connect directly to the Kubernetes API.
- Never bypass Rancher authentication.
- Never create, update, patch, or delete resources.
- Never restart, cancel, or retry PipelineRuns.
- Never execute commands inside Pods.
- Never access Kubernetes Secrets.
- Never print Rancher credentials.
- Never expose authentication headers.
- Never disable TLS verification.
- Never use arbitrary user-provided API hosts.
- Treat pipeline logs and parameters as potentially sensitive.

## Example Requests

Use this skill when the user asks:

- "Show me the latest Tekton PipelineRuns."
- "Are my Tekton pipelines green?"
- "What is currently running?"
- "Why did my last deployment fail?"
- "Investigate the latest failed PipelineRun."
- "Find the failed TaskRuns."
- "Show the error logs from the last build."
- "Check whether the staging deployment succeeded."
- "What happened during the last PipelineRun?"
- "Which Tekton tasks are failing?"

---

## How to improve this skill

This file is a cached copy of `claude-skills/skills/tekton-pipelineruns/SKILL.md` in
[beyond-scale-group/bsg-stack](https://github.com/beyond-scale-group/bsg-stack).
That repo is the single source of truth — `~/.claude/skills/tekton-pipelineruns/SKILL.md` is
overwritten every time the BSG install flow runs.

If the user asks you to improve, fix, or extend this skill, do **not** edit
the local file. Instead:

1. `gh repo clone beyond-scale-group/bsg-stack` (or work in an existing clone)
2. Edit `claude-skills/skills/tekton-pipelineruns/SKILL.md` on a feature branch
3. Open a pull request against `main`

Bug reports and ideas without a fix → open an issue on the same repo.
