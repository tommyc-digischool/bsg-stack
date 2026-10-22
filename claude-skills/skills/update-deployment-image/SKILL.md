---
name: update-deployment-image
description: >
  Mutation step of the Rancher deployment workflow: updates ONE approved
  container image in ONE specific Deployment of the Rancher preproduction
  cluster with a strategic-merge PATCH pinned to the latest
  resourceVersion. Revalidates the target, re-runs the Tekton
  "Preproduction Deployment Gate" immediately before PATCH, preserves the
  rest of the workload, then hands off to `verify-rollout`. Normally
  invoked by `update-workload-version`, which collects approval first. Use
  only with an exact, user-approved target and image reference.
model: sonnet
---

# Update Deployment Image

> Part of the Rancher/Tekton preproduction deployment skill set (Rancher API
> v2.10.2). Requires: `rancher-config`, `resolve-deployment-target`,
> `tekton-pipelineruns`, `verify-rollout`. Setup and prerequisites:
> [`update-workload-version/references/deployment-guide.md`](../update-workload-version/references/deployment-guide.md).

## Inputs

- namespace_name: required
- deployment_name: required
- container_name: required
- expected_current_image: required
- new_image: required, complete image reference
- deployment_uid: required
- resource_version: required
- user_approval: required
- target_environment: fixed to preproduction by rancher-config
- tekton_gate_evidence: required for every deployment; application and
  image/tag linkage, required PipelineRun identities, and gate result

## Preconditions

1. Require explicit approval for the exact target and new image.
2. Reject missing or blank inputs.
3. Run rancher-config.
4. GET the selected Deployment:

   `GET {kubernetes_api_base_url}/apis/apps/v1/namespaces/{namespace_name}/deployments/{deployment_name}`

5. Verify its name, namespace, and UID.
6. Verify that the named container still uses expected_current_image.
7. Verify that the target container exists.
8. Use the latest resourceVersion from this GET request.
9. If the target changed since discovery or approval, stop and
   ask the user to review the updated change.

## Preproduction deployment gate

Apply this gate even when this skill is invoked directly.

The configured Rancher cluster is always preproduction. Apply this gate
to every deployment, regardless of namespace names or metadata.

Follow the `tekton-pipelineruns` skill's "Preproduction
Deployment Gate" procedure for this exact application and new_image.
Immediately before PATCH, refresh the matching runs and recheck the
gate, including newly created runs for the same artifact. Earlier
evidence or user approval alone does not satisfy this precondition.

PATCH is allowed only when the gate returns `ready`. Otherwise stop
and report the blocking PipelineRuns or missing evidence without
modifying the Deployment. Never bypass this gate on approval alone.

## Image validation

1. Accept either a complete image reference or a version that
   can be safely combined with a known image repository.
2. Do not guess the repository.
3. Prefer explicit version tags or immutable digests.
4. Do not silently replace a missing version with latest.
5. Show the resolved image reference before requesting approval.

## Mutation

Perform only after all preconditions and the mandatory Tekton gate pass.

`PATCH {kubernetes_api_base_url}/apis/apps/v1/namespaces/{namespace_name}/deployments/{deployment_name}`

Headers:

`Content-Type: application/strategic-merge-patch+json`

Body:

```json
{
  "metadata": {
    "resourceVersion": "<latest_resource_version>"
  },
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "<container_name>",
            "image": "<new_image>"
          }
        ]
      }
    }
  }
}
```

Replace all placeholders with validated values.

## Response handling

- 2xx: record the returned state; this does not prove rollout success.
- 401 or 403: report authentication or authorization failure.
- 404: stop and re-discover the target.
- 409: re-read the Deployment and require renewed review
  if the proposed change or target state has changed.
- Other errors: report the HTTP status and sanitized response.

## Output

- previous_image
- requested_image
- update_result
- namespace_name
- deployment_name
- container_name

## Safety

- Modify only the selected container's image.
- Preserve other containers, probes, environment variables,
  resources, replicas, and rollout strategy.
- Never disable TLS verification.
- Never log credentials.
- Never automatically roll back or retry indefinitely.
- Call verify-rollout after a successful patch.

---

## How to improve this skill

This file is a cached copy of `claude-skills/skills/update-deployment-image/SKILL.md` in
[beyond-scale-group/bsg-stack](https://github.com/beyond-scale-group/bsg-stack).
That repo is the single source of truth — `~/.claude/skills/update-deployment-image/SKILL.md` is
overwritten every time the BSG install flow runs.

If the user asks you to improve, fix, or extend this skill, do **not** edit
the local file. Instead:

1. `gh repo clone beyond-scale-group/bsg-stack` (or work in an existing clone)
2. Edit `claude-skills/skills/update-deployment-image/SKILL.md` on a feature branch
3. Open a pull request against `main`

Bug reports and ideas without a fix → open an issue on the same repo.
