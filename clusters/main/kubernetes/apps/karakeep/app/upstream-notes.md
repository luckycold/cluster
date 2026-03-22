# Karakeep Migration Notes

Upstream reference evaluated: <https://github.com/karakeep-app/karakeep/tree/main/kubernetes>

This app is intentionally implemented as standalone manifests in this repo (not a direct upstream base) for a lower-risk migration away from TrueCharts.

- Upstream canonical manifests split web/chrome/meilisearch into separate Deployments and Services.
- This repo now tracks the upstream raw Kubernetes manifests by release ref in `kustomization.yaml` and layers local patches on top so future upstream drift is easier to compare.
- PVC names are now generic so a fresh cluster can bind the expected claims without carrying the old TrueCharts naming forward:
  - `karakeep-data`
  - `karakeep-meilisearch-data`
- The restic repository paths still keep the old names for backup continuity:
  - `hoarder-app-template-data-volsync-pcloud`
  - `hoarder-app-template-meili-data-volsync-pcloud`
- The public hostname is `keep.${DOMAIN_0}`, and the Authelia OIDC client id is now `karakeep` even though it still reuses the existing hoarder-named client secret material for now.

Result: the GitOps app path and namespace use `karakeep`, the workload shape stays close to upstream, the PVCs are generic for easier bring-up, and the restic repository paths stay backward-compatible.
