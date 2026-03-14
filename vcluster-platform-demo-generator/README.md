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
  - Bootstraps Argo CD application wiring to per-demo GitHub repository and webhook flow.
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
