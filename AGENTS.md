# Repository Operational Notes (High-Value Only)

## Validation Reality
CI is intentionally minimal (placeholder workflow), so local validation is required before PRs:
- `kustomize build clusters/main/kubernetes`
- `flux diff ks <name> --path <path>`
- `kubectl apply --server-side --dry-run=client -k <path>` when a target cluster/context is available
- `talosctl validate --file clusters/main/talos/talconfig.yaml` for Talos changes

## Secrets Invariants
- Keep the Age private key local as `age.agekey`; never commit raw secrets.
- Files matched by `.sops.yaml` must remain encrypted.
- Run `sops updatekeys` whenever SOPS recipients change.

## Media Namespace Migration Playbook
Use this checklist when relocating "arr" workloads (or any HelmRelease) into the shared `media` namespace:

1. **Pre-work (cluster state)**
   - `flux suspend hr <app> -n <legacy-ns>` to stop Flux from racing the move.
   - `kubectl scale deployment <app> -n <legacy-ns> --replicas=0` so the PVC can be detached safely.
   - Remove ingress conflicts: `kubectl delete ingress <app> -n <legacy-ns>` (or equivalent Traefik routes).

2. **Clone persistent data**
   - Clone the Longhorn volume that backs `<app>-config` and create a PVC in `media` with the same claim name. (Namespace must already exist; apply `clusters/main/kubernetes/media` once per cluster.)
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
   - Confirm: `flux get helmreleases -n media`, `kubectl get pods -n media -l app.kubernetes.io/instance=<app>`, and `kubectl exec ... -- ls /config`.

5. **Cleanup**
   - Delete the legacy namespace once satisfied: `kubectl delete namespace <legacy-ns>`.
   - Re-enable chart-managed ingress if a manual one was applied during debugging.

### Migration Gotchas
- **Namespace prerequisites:** the shared `media` namespace must exist *before* Flux reconciles a moved app (`kubectl apply -k clusters/main/kubernetes/media`).
- **Ingress conflicts:** remove the old ingress or Flux will fail validation when the hostname is already claimed.
- **Nginx admission cache:** if Flux still reports `host "<fqdn>" is already defined`, delete the HelmRelease and let Flux recreate it, or recycle the nginx ingress controllers (`kubectl scale deploy/nginx-{internal,external}-controller -n nginx --replicas=0 && ... --replicas=1`) to flush the webhook cache. Re-run `flux reconcile ks <app>` afterwards.
- **Ingress webhook recreation:** deleting the HelmRelease may drop the validating webhook; re-run the nginx Helm release (`flux reconcile helmrelease nginx-internal -n nginx`) to restore it once workloads are healthy.
- **PVC binding:** for statically reattached Longhorn volumes, clear the PV's `claimRef` if needed before recreating the PVC; otherwise the new claim stays Pending.
- **Avoid manual HelmRelease apply:** let Flux create the release from Git so `${...}` placeholders resolve correctly.
- **Don't forget suspension:** if the legacy HelmRelease is not suspended, Flux may immediately recreate resources in the old namespace while you migrate.
- **LoadBalancer IP drift:** MetalLB may hand out a new IP after reconciliation; update DNS or client configs (for example, Jellyfin UI) if the VIP changes.

## Collaboration Note
This file is shared between Luke and OpenCode. OpenCode may propose or apply updates, but must always explicitly notify Luke whenever this file is changed.
