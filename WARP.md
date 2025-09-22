# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

Quick commands and workflows (safe defaults)
- Safety first (commit before operations)
  - Always save a commit before making or reconciling changes:
    ```
    git add -A
    git commit -m "savepoint: <context>"
    git push
    ```
- No unit test framework
  - This repo does not use a unit test framework. Validate changes with kustomize/flux commands below.
- kustomize (read/validate only)
  - Whole cluster (entry point):
    ```
    kustomize build ./clusters/main/kubernetes | head -n 50
    ```
  - Single app overlay (example: Authelia):
    ```
    kustomize build ./clusters/main/kubernetes/apps/authelia/app | head -n 50
    ```
  - Note: ${VAR} substitution happens in Flux postBuild at reconcile time; plain kustomize will show unsubstituted ${…}.
- Flux (inspect and reconcile)
  - List what Flux knows:
    ```
    flux get kustomizations -n flux-system
    flux get sources git -n flux-system
    flux get sources oci -n flux-system
    flux get sources helm -n flux-system
    ```
  - Trigger a safe reconcile for a specific Kustomization (replace <name> with one from "flux get kustomizations"):
    ```
    flux reconcile kustomization <name> -n flux-system --with-source
    ```
  - Refresh a specific Source:
    ```
    flux reconcile source git <name> -n flux-system
    flux reconcile source oci <name> -n flux-system
    flux reconcile source helm <name> -n flux-system
    ```
- Secrets (encrypt/decrypt with clustertool; sops as fallback)
  - Preferred (per repo convention):
    ```
    # Encrypts and decrypts entire repo
    clustertool encrypt
    clustertool decrypt
    ```
  - Fallback with sops (ensure SOPS_AGE_KEY_FILE points to your age key; see .sops.yaml/devcontainer settings):
    ```
    export SOPS_AGE_KEY_FILE=~/.config/age/key.txt  # adjust if your setup differs
    sops -e -i clusters/main/kubernetes/apps/<app>/app/secret.yaml
    ```
  - Encryption rules and recipients are defined in .sops.yaml.
- Git workflow reminders
  - Before and after operations: commit and push. Flux will reconcile from Git.
  - Avoid kubectl apply/delete for Flux-managed resources; prefer reconcile via Flux.

Architecture overview (GitOps big picture)
- GitOps stack and bootstrap
  - FluxCD manages desired state; installation manifests are sourced via an OCIRepository (repositories/oci/flux-manifests.yaml) and applied by local Kustomizations under clusters/main/kubernetes/flux-system/flux/*.
  - TrueCharts ClusterTool is used operationally (Talos + Flux wrapper).
- Repositories/
  - Holds GitRepository, HelmRepository (OCI) and OCIRepository definitions that feed Flux.
- Entry point: clusters/main/kubernetes
  - The top-level Kustomization composes multiple stacks: apps, core, system, kube-system, networking, and flux-system.
- App pattern
  - Each app lives under clusters/main/kubernetes/<layer>/…/<app>/app with:
    - helm-release.yaml, namespace.yaml, kustomization.yaml
  - A sibling ks.yaml defines the Flux Kustomization that points at that app path.
- Flux entry patches and substitution
  - flux-entry.yaml (and repositories/flux-entry.yaml) apply common patches enabling postBuild variable substitution and SOPS decryption on Kustomizations.
  - Variable substitution (${VAR}) is performed by Flux at reconcile time. Local kustomize build will not substitute values.
- Config, secrets, and SOPS
  - Cluster-wide parameters and secrets are centralized (e.g., a clustersettings ConfigMap/Secret file referenced with ${…}).
  - .sops.yaml defines encryption rules and recipients for secrets. Use clustertool encrypt by default; sops is available as a fallback.
- Talos and upgrades
  - Talos cluster configuration lives in clusters/main/talos (talconfig.yaml and generated secrets).
  - System upgrade plans live under clusters/main/kubernetes/core/system-upgrade-controller-plans.
- Authentication and app ecosystem
  - Authelia (apps/authelia) is a central OIDC provider; multiple apps integrate with it via ConfigMap + Secret.
- Data services and backups
  - CloudNativePG (CNPG) operates Postgres clusters; some HelmReleases reference CNPG.
  - VolSync credentials and settings appear in cluster config for backups.
- Automation
  - Renovate is configured for dependency updates with grouping and selective automerge rules.
  - A placeholder GitHub Actions workflow and an automerge workflow are included.
- Developer environment
  - The devcontainer and VS Code settings include sops/age integration to streamline encryption/decryption.

Notes and pitfalls
- Flux substitution vs local builds
  - Expect ${…} placeholders when running kustomize locally; actual values are rendered by Flux during reconcile. Use "flux get …" to inspect state and "flux reconcile …" to trigger re-syncs.
- Secret handling
  - Use clustertool encrypt for any new/changed secret files so they match .sops.yaml rules; commit the encrypted files.
- Adding a new app
  - Create clusters/main/kubernetes/apps/<app>/app with namespace.yaml, helm-release.yaml, kustomization.yaml (mirror an existing app).
  - Add a sibling ks.yaml pointing Flux at that app path.
  - Ensure the parent Kustomization in clusters/main/kubernetes includes or references the new app Kustomization as per the existing pattern.
  - Commit and push, then reconcile the relevant Kustomization:
    ```
    flux reconcile kustomization <parent-or-app-name> -n flux-system --with-source
    ```
- Everything is synced via Flux
  - All workloads (including apps like mealie) should flow through Git and Flux. Avoid manual changes; prefer Git + reconcile.
- Safe operations only
  - Focus on read/validate/reconcile (kustomize build, flux get, flux reconcile). Avoid destructive commands (e.g., kubectl delete) in routine workflows.
