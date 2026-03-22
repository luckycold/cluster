# Karakeep Migration Notes

Upstream reference evaluated: <https://github.com/karakeep-app/karakeep/tree/main/kubernetes>

This app is intentionally implemented as standalone manifests in this repo (not a direct upstream base) for a lower-risk migration away from TrueCharts.

- Upstream canonical manifests split web/chrome/meilisearch into separate Deployments and Services.
- This repo now tracks the upstream raw Kubernetes manifests by release ref in `kustomization.yaml` and layers local patches on top so future upstream drift is easier to compare.
- Legacy storage and backup identifiers stay in place where they affect restore continuity:
  - `hoarder-app-template-data-longhorn`
  - `hoarder-app-template-meili-data-longhorn`
  - `hoarder-app-template-data-volsync-pcloud`
  - `hoarder-app-template-meili-data-volsync-pcloud`
- The public hostname is `keep.${DOMAIN_0}`, while the Authelia OIDC client id remains `hoarder` so the URL can improve without changing the external login contract in the same step.

Result: the GitOps app path and namespace use `karakeep`, the workload shape stays close to upstream, and the PVC names plus restic repository paths stay backward-compatible for safer cutover.
