# Karakeep Migration Notes

Upstream reference evaluated: <https://github.com/karakeep-app/karakeep/tree/main/kubernetes>

This app is intentionally implemented as standalone manifests in this repo (not a direct upstream base) for a lower-risk migration away from TrueCharts.

- Upstream canonical manifests split web/chrome/meilisearch into separate Deployments and Services.
- This repo now follows that split deployment shape so future upstream drift is easier to compare.
- Legacy storage and backup identifiers stay in place where they affect restore continuity:
  - `hoarder-app-template-data-longhorn`
  - `hoarder-app-template-meili-data-longhorn`
  - `hoarder-app-template-data-volsync-pcloud`
  - `hoarder-app-template-meili-data-volsync-pcloud`
- The public hostname remains `h.${DOMAIN_0}` and the Authelia OIDC client id remains `hoarder` so the external login contract does not change in the same step.

Result: the GitOps app path, namespace, workload resources, and internal service names use `karakeep`, while the PVC names and restic repository paths stay backward-compatible for safer cutover.
