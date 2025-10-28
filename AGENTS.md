# Repository Guidelines

## Project Structure & Module Organization
Configuration lives under `clusters/main`. Talos bootstrap files sit in `clusters/main/talos`; generated artifacts stay in `generated/`, and custom patches in `patches/`. Kubernetes manifests are grouped by domain inside `clusters/main/kubernetes`, where each workload folder keeps a Flux `ks.yaml` plus an `app/` subtree with the namespace, `helm-release.yaml`, and related ConfigMap or Secret overlays. Shared Git, Helm, and OCI sources are declared once in `repositories/`.

## Build, Test, and Development Commands
Run `kustomize build clusters/main/kubernetes` before every commit to catch YAML or schema errors. Use `flux diff ks authelia --path clusters/main/kubernetes/apps/authelia/app` (swap in your target path) to preview Flux changes. Secrets or config edits should be encrypted again with `sops --encrypt --in-place <file>` after editing via `sops <file>`. For Talos changes, validate with `talosctl validate --file clusters/main/talos/talconfig.yaml`.

## Coding Style & Naming Conventions
Follow YAML with 4-space indentation and lowercase keys. Kustomization manifests use `ks.yaml`; Helm releases stay in `helm-release.yaml`; SOPS-managed payloads end with `.secret.yaml` or `values.yaml`. Keep resource blocks alphabetized where practical, use blank lines sparingly for legibility, and avoid mixing secrets and plain config in the same file.

## Testing Guidelines
CI is intentionally light—the placeholder workflow only gates PRs. Rely on local validation: `kustomize build`, targeted `flux diff`, and `kubectl apply --server-side --dry-run=client -k <path>` against a staging cluster when available. For encrypted manifests, confirm `sops -d <file> | kubeconform --strict` when schemas exist, and note any manual checks (helm tests, smoke deploys) in the PR.

## Commit & Pull Request Guidelines
Match the existing Conventional Commit style (`fix:`, `chore(flux):`, etc.) and scope changes to the area touched. Keep commits focused on one logical change so Flux diffs stay readable. Pull requests must describe intent, reference related issues, and summarize validation (commands run, screenshots for UI workloads, or dashboard links). Flag secret rotations and call out new Age recipients. Wait for the placeholder workflow and automerge status before merging.

## Secrets & Configuration
Store the Age private key locally as `age.agekey`; never commit raw secrets. Files covered by `.sops.yaml` must remain encrypted—run `sops updatekeys` whenever recipients change. For new services, create separate `.secret.yaml` manifests instead of embedding secrets in `helm-release.yaml`, and reference the chart value that consumes them.

## Media Namespace Migration Playbook
Use this checklist when relocating “arr” workloads (or any HelmRelease) into the shared `media` namespace:

1. **Pre-work (cluster state)**
   - `flux suspend hr <app> -n <legacy-ns>` to stop Flux from racing the move.
   - `kubectl scale deployment <app> -n <legacy-ns> --replicas=0` so the PVC can be detached safely.
   - Remove ingress conflicts: `kubectl delete ingress <app> -n <legacy-ns>` (or equivalent Traefik routes).

2. **Clone persistent data**
   - Clone the Longhorn volume that backs `<app>-config` and create a PVC in `media` with the same claim name. (Namespace must already exist—apply `clusters/main/kubernetes/media` once per cluster.)
   - Verify with `kubectl get pvc <app>-config -n media` before touching Git.

3. **Git changes**
   - `git mv clusters/main/kubernetes/apps/<app> clusters/main/kubernetes/media/<app>`.
   - Update `helm-release.yaml`:
     - Set `metadata.namespace: media`.
     - Add `persistence.config.existingClaim: <app>-config` and keep the existing Volsync block.
   - Drop the per-app `namespace.yaml` from `app/kustomization.yaml`.
   - Fix Flux wiring: update `<app>/ks.yaml` path, add `media/<app>/ks.yaml` to `media/kustomization.yaml`, and remove `<app>/ks.yaml` from `apps/kustomization.yaml`.
   - Run `kustomize build clusters/main/kubernetes` before committing.

4. **Reconcile**
   - Commit (`chore(media): relocate <app> to shared namespace`), push, then:
     - `flux reconcile source git cluster`
     - `flux reconcile ks media`
     - `flux reconcile ks <app>`
   - Confirm: `flux get helmreleases -n media`, `kubectl get pods -n media -l app.kubernetes.io/instance=<app>`, and `kubectl exec … -- ls /config`.

5. **Cleanup**
   - Delete the legacy namespace once satisfied: `kubectl delete namespace <legacy-ns>`.
   - Re-enable chart-managed ingress if a manual one was applied during debugging.

### Gotchas
- **Namespace prerequisites:** the shared `media` namespace must exist *before* Flux reconciles a moved app (`kubectl apply -k clusters/main/kubernetes/media`).
- **Ingress conflicts:** remove the old ingress or Flux will fail validation when the hostname is already claimed.
- **Nginx admission cache:** if Flux still reports `host "<fqdn>" is already defined`, delete the HelmRelease and let Flux recreate it, or recycle the nginx ingress controllers (`kubectl scale deploy/nginx-{internal,external}-controller -n nginx --replicas=0 && ... --replicas=1`) to flush the webhook cache. Re-run `flux reconcile ks <app>` afterwards.
- **Ingress webhook recreation:** deleting the HelmRelease may drop the validating webhook; re-run the nginx Helm release (`flux reconcile helmrelease nginx-internal -n nginx`) to restore it once workloads are healthy.
- **PVC binding:** for statically reattached Longhorn volumes, clear the PV’s `claimRef` if needed before recreating the PVC; otherwise the new claim stays Pending.
- **Avoid manual HelmRelease apply:** let Flux create the release from Git so `${…}` placeholders resolve correctly.
- **Don’t forget suspension:** if the legacy HelmRelease isn’t suspended, Flux may immediately recreate resources in the old namespace while you migrate.
- **LoadBalancer IP drift:** MetalLB may hand out a new IP after reconciliation; update DNS or client configs (e.g., Jellyfin UI) if the VIP changes.
