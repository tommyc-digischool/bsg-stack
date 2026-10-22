---
name: resolve-deployment-target
description: >
  Narrows `discover-rancher-resources` candidates down to exactly one
  Rancher Deployment + container, asking the user to pick from a numbered
  list when several plausible targets remain, then re-reads the
  Deployment to return its UID and latest resourceVersion. Never picks an
  ambiguous target on its own; selection is not approval to mutate.
  Building block of `update-workload-version`. Use when a deployment
  request matches several workloads, or the user asks "which of these
  deployments is the right one?", "lequel de ces déploiements ?".
model: sonnet
---

# Resolve Deployment Target

> Part of the Rancher/Tekton preproduction deployment skill set (Rancher API
> v2.10.2). Requires: `discover-rancher-resources`. Setup and prerequisites:
> [`update-workload-version/references/deployment-guide.md`](../update-workload-version/references/deployment-guide.md).

## Inputs

- user_request: required
- candidates: required, from discover-rancher-resources

## Procedure

1. Review and deduplicate candidates using:
   cluster ID, namespace, Deployment name, and container name.
2. If there are no candidates, ask for an application name,
   team, repository, or other useful identifying information.
3. If one candidate has strong, unambiguous evidence, present
   its identity and current image for confirmation.
4. If multiple plausible candidates remain, show a concise
   numbered list containing:
   - application or Deployment name
   - namespace
   - current image
   - available environment clues
5. Ask the user to select the intended workload.
6. Do not proceed while the target remains ambiguous.
7. After selection, GET the individual Deployment again.
8. Verify its name, namespace, UID, and container list.
9. Return the exact target and the latest resourceVersion.

## Output

- cluster_id
- cluster_name
- namespace_name
- deployment_name
- container_name
- current_image
- deployment_uid
- resource_version
- target_confirmed

## Safety

- Never choose an ambiguous target automatically.
- Never update all matching resources by default.
- Treat user selection as target identification, not approval
  to perform a destructive or production-impacting mutation.
- Revalidate the target immediately before updating.

---

## How to improve this skill

This file is a cached copy of `claude-skills/skills/resolve-deployment-target/SKILL.md` in
[beyond-scale-group/bsg-stack](https://github.com/beyond-scale-group/bsg-stack).
That repo is the single source of truth — `~/.claude/skills/resolve-deployment-target/SKILL.md` is
overwritten every time the BSG install flow runs.

If the user asks you to improve, fix, or extend this skill, do **not** edit
the local file. Instead:

1. `gh repo clone beyond-scale-group/bsg-stack` (or work in an existing clone)
2. Edit `claude-skills/skills/resolve-deployment-target/SKILL.md` on a feature branch
3. Open a pull request against `main`

Bug reports and ideas without a fix → open an issue on the same repo.
