# Rancher/Tekton deployment skills — setup and usage

Rancher and these skills are used **only for staging/preproduction**.
Production deployments follow a separate GitOps process (see
[Production is out of scope](#production-is-out-of-scope)).

The skill set updates an application's container image in the Rancher
cluster named `local` (preproduction):

| Skill | Role | Mutates? |
|-------|------|----------|
| `update-workload-version` | Entry point — orchestrates the whole flow | via `update-deployment-image` |
| `rancher-config` | Rancher host, credentials, `local` cluster resolution | No |
| `discover-rancher-resources` | Find namespaces / Deployments / containers for an app | No |
| `resolve-deployment-target` | Narrow candidates to one Deployment + container | No |
| `tekton-pipelineruns` | Build status, failure diagnosis, Preproduction Deployment Gate | No |
| `update-deployment-image` | PATCH one container image after approval | **Yes** |
| `verify-rollout` | Confirm the new image is running and healthy | No |

Start with `update-workload-version`; it coordinates the other six.
`tekton-pipelineruns` can also be used on its own to check builds.

## Install

The skills ship with the BSG catalog: the BSG installer caches them into
`~/.claude/skills/` (see `claude-skills/INSTALL.md` in
[beyond-scale-group/bsg-stack](https://github.com/beyond-scale-group/bsg-stack)).
No manual copy is needed. Start a new Claude Code session after install.

## Before using

Export these variables in the shell **before** launching Claude Code
(or provide them through a secret provider the agent can access):

```sh
export RANCHER_URL="https://rancher.example.com"  # your Rancher host, no trailing slash
export RANCHER_ACCESS_KEY="<access-key>"          # Rancher API key — access part
export RANCHER_SECRET_KEY="<secret-key>"          # Rancher API key — secret part
```

- Never paste credentials into prompts and never commit them.
- The Rancher account behind the API key needs read access to the
  relevant resources. Updating an image also requires permission to
  update the intended Deployment.
- For `tekton-pipelineruns`, the identity needs `get` and `list` on
  PipelineRuns and TaskRuns, plus `get` on Pods and Pod logs in the
  `tekton-pipelines` namespace.
- The agent must be allowed to make HTTPS requests to `$RANCHER_URL`.
  The skills contain instructions only; the agent needs an HTTP client or
  shell access (e.g. `curl`) to perform them.
- Before updating an image, its Tekton build must have finished
  successfully — the workflow enforces this through the
  `tekton-pipelineruns` Preproduction Deployment Gate.

## Use

### Check Tekton builds

```text
/tekton-pipelineruns Show the latest 5 PipelineRuns.
/tekton-pipelineruns Investigate why PipelineRun <name> failed and show relevant error logs.
```

This skill is read-only and operates only in the `tekton-pipelines`
namespace through authenticated Rancher HTTP APIs. It reuses
`rancher-config`; it does not use `kubectl` or the Tekton Dashboard. It
reports run status, investigates failed TaskRuns and steps, and retrieves
relevant log excerpts. It never restarts, cancels, or retries builds. A
successful build does not by itself confirm a healthy deployment.

### Update a workload

```text
/update-workload-version Update <application> to version <tag> in preproduction.
```

You can also provide a complete image reference. The agent discovers the
namespace, Deployment, and container, asks you to choose if the target is
ambiguous, checks the Tekton gate, then requests approval for the exact
target and image before changing anything. It verifies the rollout
afterward and reports failures without automatically rolling back.

For read-only discovery, ask for `discover-rancher-resources` to find an
application and show its current image.

## Manual equivalent in the Rancher UI

What `update-workload-version` automates, for reference:

1. Sign in to Rancher and select the preproduction cluster.
2. Open **Workloads** > **Deployments**.
3. Find the Deployment, open its three-dot menu, select **Edit Config**.
4. After confirming the Tekton build succeeded, change only the image
   version and save.
5. Back on the Deployments list, the status briefly turns red, then green
   once the new image is running.

Tip: in Rancher profile preferences, set **Login Landing Page** to the
cluster named **Local** so Rancher opens preproduction after sign-in.

## Production is out of scope

These skills never deploy to production and must not be retargeted to a
production host or cluster. Production releases go through the team's
GitOps deployments repository:

1. A notification is posted in the team's release channel for each new
   final release tag (e.g. `1.x.x`) of projects linked to a merge request
   in the GitOps deployments repository.
2. The merge request must be approved by at least two people before it is
   merged into `master`.
3. Once merged into `master`, the new version is deployed to production
   automatically.

Ask your team for the GitOps repository and release-channel links — they
are internal and intentionally not listed in this public catalog.
