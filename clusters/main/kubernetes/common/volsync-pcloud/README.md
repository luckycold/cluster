# VolSync pCloud Pattern

Use this pattern when adding a pCloud-backed restic repository for an app PVC.

What is shared:

- pCloud hostname
- pCloud token JSON
- remote root path
- repo subpath
- retention policy

Those shared values live in:

- `clusters/main/clusterenv.yaml`
- `clusters/main/kubernetes/flux-system/flux/clustersettings.secret.yaml`

What stays per app/PVC:

- Secret name
- `RESTIC_REPOSITORY`
- `ReplicationSource` name
- `sourcePVC`

Why per app secrets still exist:

- Every restic repository path must stay unique.
- A single shared Secret like `bw-auth-token` would point multiple PVCs at the same repository.

Per-app secret pattern:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-config-volsync-pcloud
  namespace: app
type: Opaque
stringData:
  RESTIC_PASSWORD: <shared backup password>
  RESTIC_REPOSITORY: rclone:pcloud:/Server Backups/Cluster/restic/app-config-volsync-pcloud
  RCLONE_CONFIG_PCLOUD_TYPE: pcloud
  RCLONE_CONFIG_PCLOUD_HOSTNAME: api.pcloud.com
  RCLONE_CONFIG_PCLOUD_TOKEN: <shared token json>
```

ReplicationSource pattern:

```yaml
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: app-config-pcloud
spec:
  sourcePVC: app-config-longhorn
  trigger:
    manual: pcloud-test-<date>
  restic:
    repository: app-config-volsync-pcloud
    copyMethod: Snapshot
    pruneIntervalDays: 7
    cacheCapacity: 10Gi
    retain:
      daily: 3
      weekly: 4
      monthly: 2
      within: 30d
    accessModes:
      - ReadWriteOnce
    moverSecurityContext:
      runAsUser: 0
      runAsGroup: 0
      fsGroup: 568
```

Current reference apps:

- `clusters/main/kubernetes/apps/freshrss/app/`
- `clusters/main/kubernetes/apps/hoarder/app/`

Rollout checklist:

1. Add the SOPS-encrypted pCloud Secret for the PVC.
2. Add the pCloud `ReplicationSource`.
3. Add both files to the app `kustomization.yaml`.
4. Commit and push first.
5. Reconcile the app kustomization.
6. Trigger one manual sync and verify `status.lastManualSync` updates.
