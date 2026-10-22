---
name: discover-rancher-resources
description: >
  Read-only discovery of namespaces, Deployments, and containers in the
  Rancher preproduction cluster (`local`) from a natural-language
  application description, via the authenticated Rancher Kubernetes proxy
  API. Ranks candidates by evidence (names, labels, image repository) and
  never guesses. Building block of `update-workload-version`; also usable
  alone. Use when the user asks "which deployment runs <app>?", "what image
  is <app> running in preprod?", "find <app> in Rancher", "quelle version
  tourne en préprod ?", "trouve le déploiement de <app>".
model: sonnet
---

# Discover Rancher Resources

> Part of the Rancher/Tekton preproduction deployment skill set (Rancher API
> v2.10.2). Requires: `rancher-config`. Setup and prerequisites:
> [`update-workload-version/references/deployment-guide.md`](../update-workload-version/references/deployment-guide.md).

## Inputs

- user_request: required, natural-language description
- application_hint: optional
- namespace_hint: optional
- environment_hint: optional
- image_hint: optional

## Procedure

1. Run rancher-config and use its returned Kubernetes API base URL and resolved cluster context. Authenticate all Rancher/Kubernetes API requests using HTTP Basic Authentication with the Rancher Access Key as the username and Secret Key as the password, loaded securely from the configured secret provider or `RANCHER_ACCESS_KEY` and `RANCHER_SECRET_KEY` environment variables. Never use the Secret Key alone as a Bearer token.
2. Extract the intended application, requested version, and any available business or environment hints from user_request.
3. List namespaces:

   `GET {kubernetes_api_base_url}/api/v1/namespaces`

4. Exclude namespaces that are terminating.
5. For each eligible namespace, list Deployments:

   `GET {kubernetes_api_base_url}/apis/apps/v1/namespaces/{namespace}/deployments`

6. Inspect each candidate's:
   - metadata.name
   - metadata.namespace
   - metadata.labels
   - metadata.annotations
   - metadata.uid
   - metadata.resourceVersion
   - spec.selector.matchLabels
   - spec.template.metadata.labels
   - spec.template.spec.containers[].name
   - spec.template.spec.containers[].image
   - status conditions and replica counts

7. Match the application hint against available evidence:
   - exact Deployment or container name
   - application labels and annotations
   - image repository and image name
   - known service identifiers
   - namespace or team metadata
   - an optional external service catalog, if configured

8. Rank candidates by evidence strength.
9. Return plausible matches, including their matching evidence.
10. If no candidates match, report that and ask for a useful identifying detail instead of guessing.

## Output

Return candidates with:
- cluster_id
- cluster_name
- namespace_name
- deployment_name
- deployment_uid
- resource_version
- containers: [{name, image}]
- matching_evidence
- confidence
- rollout_summary

## Safety

- This skill is read-only.
- Do not select the first candidate arbitrarily.
- Do not assume similar names refer to the same application.
- Do not inspect or disclose Secret contents.
- Do not claim semantic certainty from weak metadata matches.
- Do not mutate resources.
- Never hardcode, print, or expose authentication credentials.
- Do not disable TLS certificate verification.

---

## How to improve this skill

This file is a cached copy of `claude-skills/skills/discover-rancher-resources/SKILL.md` in
[beyond-scale-group/bsg-stack](https://github.com/beyond-scale-group/bsg-stack).
That repo is the single source of truth — `~/.claude/skills/discover-rancher-resources/SKILL.md` is
overwritten every time the BSG install flow runs.

If the user asks you to improve, fix, or extend this skill, do **not** edit
the local file. Instead:

1. `gh repo clone beyond-scale-group/bsg-stack` (or work in an existing clone)
2. Edit `claude-skills/skills/discover-rancher-resources/SKILL.md` on a feature branch
3. Open a pull request against `main`

Bug reports and ideas without a fix → open an issue on the same repo.
