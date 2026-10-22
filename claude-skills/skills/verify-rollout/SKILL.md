---
name: verify-rollout
description: >
  Read-only verification that a Rancher-managed Kubernetes Deployment
  actually runs the expected container image and that its rollout is
  healthy (observedGeneration, updated/ready/available replicas,
  ProgressDeadlineExceeded), polling until success, failure, or timeout.
  Called by `update-deployment-image` after a PATCH; also usable alone.
  Use when the user asks "did the preprod rollout succeed?", "is <app>
  running the new image?", "le déploiement est bien passé ?", "vérifie
  le rollout".
model: haiku
---

# Verify Deployment Rollout

> Part of the Rancher/Tekton preproduction deployment skill set (Rancher API
> v2.10.2). Requires: `rancher-config`. Setup and prerequisites:
> [`update-workload-version/references/deployment-guide.md`](../update-workload-version/references/deployment-guide.md).

## Inputs

- namespace_name: required
- deployment_name: required
- container_name: required
- expected_image: required
- timeout_seconds: optional, default 300
- poll_interval_seconds: optional, default 5

## Procedure

1. Run rancher-config.
2. Poll the Deployment using GET:

   `GET {kubernetes_api_base_url}/apis/apps/v1/namespaces/{namespace_name}/deployments/{deployment_name}`

3. Confirm that the named container's image equals expected_image.
4. Inspect:
   - metadata.generation
   - status.observedGeneration
   - spec.replicas
   - status.updatedReplicas
   - status.readyReplicas
   - status.availableReplicas
   - status.unavailableReplicas
   - status.conditions

5. Normalize omitted replica counters to zero where Kubernetes
   semantics permit.
6. Determine the desired replica count from spec.replicas,
   accounting for the default of one when omitted.
7. For a Deployment with desired replicas greater than zero,
   report success only when:
   - observedGeneration >= metadata.generation
   - updatedReplicas equals desired replicas
   - readyReplicas equals desired replicas
   - availableReplicas equals desired replicas
   - unavailableReplicas is zero
   - the selected container uses expected_image

8. For a zero-replica Deployment, report separately that the
   desired image is configured but no running replicas exist.
9. Stop when verification succeeds, the timeout expires,
   or a terminal rollout failure is detected.
10. Report ProgressDeadlineExceeded as a failed rollout.
11. On timeout, report timed_out rather than succeeded.

## Output

- rollout_status: succeeded, failed, timed_out, or zero_replicas
- expected_image
- observed_image
- desired_replicas
- updated_replicas
- ready_replicas
- relevant_conditions

## Safety

- This skill is read-only.
- Never claim success solely from the PATCH response.
- Never automatically roll back.
- Do not report zero replicas as a healthy running rollout.

---

## How to improve this skill

This file is a cached copy of `claude-skills/skills/verify-rollout/SKILL.md` in
[beyond-scale-group/bsg-stack](https://github.com/beyond-scale-group/bsg-stack).
That repo is the single source of truth — `~/.claude/skills/verify-rollout/SKILL.md` is
overwritten every time the BSG install flow runs.

If the user asks you to improve, fix, or extend this skill, do **not** edit
the local file. Instead:

1. `gh repo clone beyond-scale-group/bsg-stack` (or work in an existing clone)
2. Edit `claude-skills/skills/verify-rollout/SKILL.md` on a feature branch
3. Open a pull request against `main`

Bug reports and ideas without a fix → open an issue on the same repo.
