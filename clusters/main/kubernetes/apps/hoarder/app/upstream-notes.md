# Karakeep Migration Notes

Upstream reference evaluated: <https://github.com/karakeep-app/karakeep/tree/main/kubernetes>

This app is intentionally implemented as standalone manifests in this repo (not a direct upstream base) for low-risk migration from TrueCharts:

- Upstream canonical manifests split web/chrome/meilisearch into separate Deployments.
- Current live workload depends on localhost sidecar wiring (`BROWSER_WEB_URL=http://localhost:9222`, `MEILI_ADDR=http://localhost:7700`).
- Existing storage and backup object names must remain stable to avoid PVC/VolSync churn:
  - `hoarder-app-template-data`
  - `hoarder-app-template-meili-data`
  - `hoarder-app-template-*-volsync-minio`
  - `hoarder-app-template-*-minio`

Result: workload resources are rebranded to `karakeep` where safe (Deployment and container naming), while namespace/PVC/VolSync and externally-referenced Service/Ingress object names remain `hoarder-*` for compatibility.
