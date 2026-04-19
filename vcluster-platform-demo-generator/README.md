# vCluster Platform Demo Generator

This directory contains the manifests for a hierarchical demo setup:

1. A **management vCluster** runs **vCluster Platform (vCP)**.
2. That vCP provisions **child demo environments** as additional vCluster instances using vCluster Platform `VirtualClusterTemplate` resources.
3. Each child environment is bootstrapped with **Argo CD**, **Crossplane**, and a dedicated **GitHub template repository** workflow.

In short: **vCP running inside a vCluster creates demo vCP environments running in child vCluster instances**.

## Architecture

- **Layer 1 (host cluster):** Runs the top-level management vCluster.
- **Layer 2 (management vCluster):** Runs vCluster Platform and applies the GitOps package in `vcluster-platform-gitops/`.
- **Layer 3 (child vCluster instances):** Created by templates (`VirtualClusterTemplate`) and populated with:
  - vCluster Platform app
  - Argo CD app
  - Crossplane components/providers/compositions
  - GitHub repo automation (`DemoRepository` + webhook/app bootstrap)

## Directory Layout

- `vcluster-platform-gitops/`
  - Main GitOps package for the management vCP.
  - Includes templates, apps, project/secret definitions, and kustomization entrypoint.
- `vcluster-platform-bootstrap/`
  - Bootstrap app/template definitions used to stand up the initial demo generator management environment.
- `argocd/`
  - Argo CD apps/bootstrap manifests used by the demo flow.
- `crossplane/`
  - Crossplane providers and XRD/composition bundles for GitHub repo and connected host cluster integration.

## Key Manifests

- `vcluster-platform-gitops/kustomization.yaml`
  - Entry point for GitOps resources applied into management vCP.
- `vcluster-platform-gitops/virtual-cluster-templates/vcluster-platform-demo-template.yaml`
  - Main child environment template.
  - Defines installed apps, bootstrap objects/secrets, labels/annotations, and vCluster behavior.
- `vcluster-platform-gitops/apps/vcluster-platform.yaml`
  - vCluster Platform app definition used by child templates.
- `vcluster-platform-gitops/apps/argocd.yaml`
  - Argo CD app definition used in child templates.
- `vcluster-platform-gitops/apps/crossplane-manifests.yaml`
  - Installs/applies Crossplane bootstrap manifests.
- `vcluster-platform-gitops/apps/github-repo-argo-cd-webhooks.yaml`
  - Waits for the generated repo `replace-text` completion marker, then bootstraps Argo CD application wiring to the per-demo GitHub repository and webhook flow.
- `crossplane/vcluster-demo-repository-x/*`
  - `DemoRepository` API (XRD + Composition) for template repo provisioning/automation.

## End-to-End Flow

1. Bootstrap a management vCluster with vCP, Argo CD, and Crossplane (`vcluster-platform-bootstrap/`).
2. Apply `vcluster-platform-gitops/` resources into that management vCP.
3. Management vCP exposes templates (for example `vcluster-platform-demo`) for creating child demo environments.
4. Creating a child environment:
  - deploys vCluster Platform + Argo CD inside the child vCluster,
  - provisions required namespaces/secrets/labels,
  - triggers GitHub repo scaffolding via `DemoRepository` (Crossplane),
  - wires Argo CD to sync from the generated repo path.

## Branch-Test Repo Generation

The demo generator now supports creating child demo repos from a non-`main` branch of `vcluster-platform-demo-app-template` for testing.

### User-facing behavior

- `vcluster-platform-demo-template.yaml` exposes an `Advanced` parameter named `gitRevision`
- when `gitRevision` is `main`, the default value, the generated repo keeps the existing behavior
- when `gitRevision` is a feature or test branch, the generated repo is created from that template branch and its default branch is switched to the same value

### Crossplane flow

The `crossplane/vcluster-demo-repository-x/` package handles the sequencing:

1. `DemoRepository.spec.gitTargetRevision` receives the selected branch.
2. `Repository` creates the generated repo from the template.
3. For non-`main` cases, the composition enables copying all template branches.
4. `DefaultBranch` updates the generated repo default branch to `gitTargetRevision`.
5. After `DefaultBranch` is ready, the seed `RepositoryFile` writes `.generator-seed/replace-text-trigger.txt` on `gitTargetRevision` with the existing commit message `seed: [run replace-text]`.
6. That seed commit triggers the generated repo `replace-text` GitHub Actions workflow, which renders `{REPLACE_GIT_TARGET_REVISION}` in self-repo Argo CD and Flux manifests, including repos that keep the demo content under `project/`, and commits `.generator-seed/replace-text-complete.txt` on success.
7. The `github-repo-argo-cd-webhooks` bootstrap app waits for `.generator-seed/replace-text-complete.txt` on the generated repo default branch before it creates the root `vcluster-gitops` Argo CD `Application`.

This ordering is important. The default branch must be switched before the seed commit lands, otherwise the generated repo can render `main` into self-repo references even when a branch-test flow was requested.

### Template author expectations

The companion template repo `vcluster-platform-demo-app-template` should use `{REPLACE_GIT_TARGET_REVISION}` for self-repo:

- Argo CD `targetRevision`
- Flux `GitRepository.spec.ref.branch`

That keeps normal `main` repos unchanged while letting generated branch-test repos follow their selected branch automatically.

## Apply Notes

Typical apply target for the management vCP GitOps package:

```bash
kubectl apply -k vcluster-platform-demo-generator/vcluster-platform-gitops
```

Bootstrap resources can be applied separately from `vcluster-platform-bootstrap/` depending on your environment and Loft workflow.

## Required External Inputs

These resources are expected to be populated by project/space secrets or platform automation:

- OIDC secret(s) for vCP auth
- GitHub provider credentials for Crossplane
- GHCR pull credentials (`ghcr-login-secret`)
- Optional TLS/license secrets used by vCP app manifests

## Operational Notes

- Many manifests are templated using Loft variables (for example `{{ .Values.loft.virtualClusterName }}`).
- Argo CD and vCluster sync/runtime options (including optional `runtimeClassName: vnode`) are controlled in template values.
- If startup becomes slow with runtime class enabled, hook timeouts in app bootstrap scripts may need tuning.
