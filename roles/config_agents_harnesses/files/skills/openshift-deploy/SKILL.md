---
name: openshift-deploy
description: Use when deploying applications to the OpenShift cluster, iterating fast with oc, or updating real deployments through Argo CD / the GitOps repo at ~/ws/crowdsalat/ocp-gitops. Trigger keywords: oc, openshift, ocp, deploy to cluster, rollout, argo, gitops, argocd, image stream, namespace on cluster.
---

# OpenShift deploy

Argo CD (OpenShift GitOps) is the state carrier. Manifests live in the app repo; the gitops repo only registers the Argo CD Application. Autosync is never committed — toggled at runtime: **off for inner loop, on for outer loop**.

## Where manifests live
- Project repo (default) — e.g. `~/ws/crowdsalat/teddycloud-spotify-radio-shim@container/ocp`; registered in `gitops/bootstrap/apps-applicationset-external-manifests.yaml` (maps name → repoURL + path + namespace).
- Gitops repo (legacy) — teddycloud, mosquitto, openclaw, obsidian-sync, mealie under `gitops/applications/<app>/` (picked up by the `applications` ApplicationSet).

## New app (no Argo CD app yet)
1. Prefer the external-manifests pattern: add name / repoURL / path / namespace to the ApplicationSet in **`~/ws/crowdsalat/ocp-gitops`**.
2. Commit + push; Argo applies the Application automatically.

## Sync toggle (runtime, not git)
- On: `argocd app set <app> --sync-policy automated` (or UI)
- Off: `argocd app set <app> --sync-policy none` (or UI)
- Never commit `syncPolicy.automated`.

## Inner loop (fast iteration) — autosync OFF
1. Build image locally (containers skill).
2. `oc login <api-url> --web`; `oc project <namespace>`.
3. Autosync off = Argo won't self-heal:
   - `oc apply -f deployment.yaml`, or
   - `oc set image deployment/<name> <container>=<image>:<tag>`
   - check: `oc rollout status deployment/<name>`, `oc get pods`, `oc logs deployment/<name> -f`
4. Toggle autosync back **on** when done.

## Outer loop (tagged release) — autosync ON
1. Publish image tag (containers skill), e.g. `ghcr.io/<owner>/<repo>/<name>:0.7.1`.
2. Bump `image:` in the app's manifest — project repo (default) or `gitops/applications/<app>/` (legacy) — commit + push.
3. Pick up: autosync on, or `argocd app sync <app>`.
4. Watch: `argocd app get <app>`, `oc get pods`.

## Rules
- Only explicit version tags (`0.7.0`), never `:latest`.
- Non-restricted SCC: justify explicitly via the `clusterrolebinding-anyuid.yaml` pattern.