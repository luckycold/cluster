# Migration Backup Guardrails

This branch keeps the new Ubuntu + k3s cluster in pull-only mode until the migration is stable. The goal is to prevent the new cluster from overwriting, rotating, or appending to backup streams created by the old cluster.

## VolSync

### Chart-managed app backups

For app-template style `volsync` values, every `src.enabled` flag has been set to `false` while leaving `dest.enabled` enabled. This keeps restore destinations available without allowing the new cluster to push snapshots back to object storage.

Affected apps:

- `alist`
- `apprise`
- `audiobookshelf`
- `changedetection-io`
- `freshrss`
- `kavita`
- `mealie`
- `nextcloud`
- `ntfy`
- `jellyfin`
- `lidarr`
- `maintainerr`
- `notifiarr`
- `overseerr`
- `prowlarr`
- `qbittorrent`
- `radarr`
- `sonarr`
- `speakarr`
- `tautulli`

Exception during validation:

- The arr apps now back up again through explicit standalone `ReplicationSource` manifests instead of the chart-managed `volsync.src` path:
  - `lidarr`
  - `prowlarr`
  - `radarr`
  - `sonarr`
  - `readarr`
  - `speakarr`

### Standalone VolSync manifests

Every explicit `ReplicationSource` manifest has `spec.paused: true` so the object can still exist in Git without scheduling outbound syncs.

Affected manifests:

- `clusters/main/kubernetes/apps/freshrss/app/volsync.yaml`
- `clusters/main/kubernetes/apps/karakeep/app/volsync.yaml`
- `clusters/main/kubernetes/apps/lldap/app/volsync.yaml`
- `clusters/main/kubernetes/apps/minecraft/app/volsync.yaml`
- `clusters/main/kubernetes/apps/romm/app/volsync.yaml`
- `clusters/main/kubernetes/media/speakarr/app/volsync-restic-replicationsource.yaml`

`ReplicationDestination` restore objects were left intact.

Additional restore destinations were added where a standalone app had a backup source but no pull target yet:

- `clusters/main/kubernetes/apps/karakeep/app/volsync.yaml`
- `clusters/main/kubernetes/apps/lldap/app/volsync.yaml`
- `clusters/main/kubernetes/apps/romm/app/volsync.yaml`
- `clusters/main/kubernetes/media/maintainerr/app/volsync-restore.yaml`

Maintainerr restores from the existing MinIO repository path but uses its historical repository password through `VOLSYNC_BACKUP_MAINTAINERR_ENCRKEY`.

This is a real migration exception: the object-store location moved to MinIO, but the repository password did not rotate with `VOLSYNC_BACKUP_MAIN_ENCRKEY`.


Standalone sources that already had a chart-managed restore path were left as-is:

- `clusters/main/kubernetes/apps/freshrss/app/helm-release.yaml`
- `clusters/main/kubernetes/media/jellyfin/app/helm-release.yaml`
- `clusters/main/kubernetes/media/speakarr/app/helm-release.yaml`

## PostgreSQL Backups

### Raw CloudNativePG manifests

The direct CNPG cluster manifests were changed so the new cluster does not write Barman backups or WAL archives to the shared backup bucket during migration.

- removed the `spec.backup` block from the CNPG `Cluster`
- switched `bootstrap` from `initdb` to `recovery`
- added `externalClusters` definitions that point at the existing object-store backups
- set the `ScheduledBackup` resources to `suspend: true`

Affected manifests:

- `clusters/main/kubernetes/apps/authelia/app/cnpg.yaml`
- `clusters/main/kubernetes/apps/lldap/app/cnpg.yaml`

### Chart-managed CNPG backups

For chart-managed CNPG instances, the branch now uses pull-only recovery mode:

- `cnpg.main.mode: recovery`
- `cnpg.main.recovery` points at the existing backup credentials/revision
- `cnpg.main.backups.enabled: false` keeps the new cluster from writing back yet

Affected manifests:

- `clusters/main/kubernetes/apps/mealie/app/helm-release.yaml`
- `clusters/main/kubernetes/apps/nextcloud/app/helm-release.yaml`

## Re-enable Later

After cutover and validation on the new cluster:

1. re-enable VolSync push sources
2. unpause standalone `ReplicationSource` objects
3. restore CNPG backup configuration and unsuspend scheduled backups
4. confirm the old cluster is no longer writing to the same backup locations before turning the new writers back on

## Restore Workflow Notes

- Suspend risky apps before restore validation if they can create outside effects or overwhelm a single-node test cluster:
  - `qbittorrent` to avoid tracker activity
  - `minecraft` to avoid unnecessary resource pressure during bootstrap
- Suspend the affected Flux kustomizations before manual restore surgery, or commit the testing changes first, so Flux does not recreate workloads while the restore is still running.
- If the workstation kubeconfig still points at a dead or not-yet-routed VIP, use `ssh home-apps.lan.1al.cc 'sudo k3s kubectl ...'` as break-glass access until kubeconfig and networking catch up.
- Prefer `ReplicationDestination.spec.restic.destinationPVC` when an app can restore directly into its live PVC.
- For SQLite-backed apps, copy out the pre-restore database and scale the workload down before triggering the restore.
- If a restore path creates a separate destination PVC instead of writing into the live app claim, copy the restored data into the live PVC before starting the app.
- If Longhorn volumes start failing with `already mounted or mount point busy` on Ubuntu, check whether `multipathd` is claiming the block devices as `/dev/mapper/mpath*`. Disable `multipathd` on the host before doing PVC surgery, then use a fresh blank PVC plus file-level copy only when a mounted volume already needs to be rebuilt.
- Karakeep needed this extra copy step when it still lived under the Hoarder app path, because its restore objects were initially restoring into `volsync-*-dest-dest` PVCs rather than the active app PVCs.
- The arr apps proved safer to restore from their own scheduled backup zips on `/config/Backups/scheduled` than from the older chart-managed VolSync path.
- After restoring the arr apps from the native backup zips, explicit standalone VolSync sources were added so fresh backups resume from the corrected PVC contents.
- After a manual `ReplicationDestination` run, success is recorded on `status.lastSyncTime` and `status.latestMoverStatus.result`; the object may already be back in `WaitingForManual` by the time you inspect it.
