# Single-Node Ubuntu + k3s Plan

This is the working plan for moving the current single-node Talos cluster onto a single Ubuntu host running k3s, while keeping the existing GitOps layout as intact as possible.

## Current Host State

- Target host: `home-apps.lan.1al.cc`
- OS: `Ubuntu 24.04.4 LTS`
- RAM: `24 GiB`
- OS disk: `64 GiB`
- Longhorn disk: `256 GiB` secondary disk (`/dev/sdb`), still unused
- Completed base prep is recorded in `docs/base-host-changes.md`

## Host Prep Sequence

1. Expand the root logical volume to use the free space already available in the Ubuntu volume group. Done.
2. Partition and format `/dev/sdb` as a dedicated Longhorn disk. Done.
3. Mount the Longhorn filesystem at `/var/lib/longhorn` using a UUID-based `/etc/fstab` entry. Done.
4. Mirror the old node's NFS client defaults so Kubernetes NFS mounts behave like the Talos node. Done.
5. Install `k3s` as a single server with the bundled components we do not use disabled. Done.
6. Bootstrap Cilium manually before installing Flux. Done.

## k3s Install Shape

Use a config file under `/etc/rancher/k3s/config.yaml` instead of a long shell command so the setup is easier to automate later.

Planned settings:

- preserve the existing pod and service CIDRs
- use embedded etcd from day one with `cluster-init: true`
- disable `traefik`
- disable `servicelb`
- disable `metrics-server`
- disable `local-storage`
- disable flannel (`flannel-backend: none`)
- disable kube-proxy
- disable the built-in network policy controller
- set a readable kubeconfig mode for the admin user
- include the node IP and hostname as TLS SANs
- set `max-pods=250`

Applied host config:

- node name: `home-apps`
- node IP: `192.168.1.28`
- TLS SANs:
  - `home-apps`
  - `home-apps.lan.1al.cc`
  - `192.168.1.28`

## Repo Changes Needed Before Full Flux Reconcile

- Remove Talos-specific upgrade plans from `clusters/main/kubernetes/core/kustomization.yaml:1` and stop reconciling `clusters/main/kubernetes/core/system-upgrade-controller-plans/app/kubernetes.yaml:1` plus `clusters/main/kubernetes/core/system-upgrade-controller-plans/app/talos.yaml:1`.
- Remove the Talos-only system upgrade controller from `clusters/main/kubernetes/system/kustomization.yaml:1` until an Ubuntu/k3s replacement exists.
- Update Cilium bootstrap settings in `clusters/main/kubernetes/kube-system/cilium/app/helm-release.yaml:71` so it points at the k3s API endpoint instead of the old Talos-local port.
- Update cluster environment values that still point at the old Talos node IP.

## Bootstrap Order

1. Install base host packages and storage prep. Done.
2. Install `k3s` with Cilium-compatible flags. Done.
3. Install Cilium manually using the same chart version and values the repo expects, but with the k3s API endpoint. Done.
4. Verify the node becomes `Ready`. Done.
5. Pre-create namespaces for HelmRelease-only operator kustomizations before the first full Flux reconcile.
6. Install Flux.
7. Reconcile the repo after the Talos-only kustomizations are removed from this branch.

## Backup Safety

- Migration-specific backup writer guardrails are recorded in `docs/migration-backup-guardrails.md`.
- Until the new cluster is stable, the branch is configured so it can restore from backups without becoming a backup writer.

## Manual Bootstrap Notes

- Cilium was manually bootstrapped with Helm `v3.19.0`.
- The chart version used was `1.18.6`, matching `clusters/main/kubernetes/kube-system/cilium/app/helm-release.yaml:1`.
- The bootstrap values matched the repo values, except the Kubernetes API endpoint was set to `home-apps.lan.1al.cc:6443`.
- The fresh host was missing `/etc/cni/net.d`, so that directory had to be created before Cilium could write `05-cilium.conflist` and make the node schedulable.
- Flux source manifests must use `source.toolkit.fluxcd.io/v1`, not `v1beta2`, on this cluster.
- `repositories/git/this-repo.yaml` must point at the migration branch during parity testing.
- `clusters/main/kubernetes/kustomization.yaml` should not include `common` at the root during bootstrap, because namespace-less shared secrets like `bw-auth-token` are meant to be included by app kustomizations.
- `repositories/helm/kustomization.yaml` must include `repositories/helm/metallb.yaml`, otherwise the dedicated MetalLB chart source never exists in-cluster.
- `clusters/main/kubernetes/system/metallb/app/helm-release.yaml` must use the dedicated `metallb` HelmRepository. The old `home-ops-mirror` source returned `403 denied` for the mirrored MetalLB chart.
- These namespaces had to exist before their HelmRelease-only kustomizations would reconcile cleanly:
  - `cert-manager`
  - `cloudnative-pg`
  - `kubernetes-reflector`
  - `snapshot-controller`
- On this cluster, `ingress-nginx` came up before MetalLB, so testing had to use the node IP plus nginx NodePorts instead of normal `80/443` VIPs.
- The ingress `ADDRESS` column can show stale old-cluster IPs during migration; do not trust it as proof that the new cluster owns those VIPs.
- Useful temporary test entrypoints were:
  - internal ingress HTTPS on `https://192.168.1.28:32597`
  - external ingress HTTPS on `https://192.168.1.28:31230`
- Some early Helm installs failed because the nginx admission webhook had no ready endpoints yet; re-reconciliation after the controllers were healthy was required.
- Blocky itself can be healthy before LAN DNS is reachable. Its `blocky-dns` `LoadBalancer` service stays `<pending>` until MetalLB is healthy.
- Some `LoadBalancer` services only honored the intended IP after adding the `metallb.io/loadBalancerIPs` annotation in addition to `loadBalancerIP`.
- If MetalLB reports `can't change sharing key`, another service is already holding the same VIP; reassign or recreate the conflicting service so each VIP is unique unless explicit sharing is intended.

## Restore/Test Mode Notes

- Suspend high-risk or high-churn apps before restore validation on the new cluster:
  - `minecraft` because it can pressure a small single-node host
  - `qbittorrent` so it does not talk to trackers during parity testing
- When doing manual restore work, Flux will fight temporary cluster-side changes unless those changes are committed or the affected kustomizations are suspended first.
- Some VolSync restores can target the live PVC directly with `destinationPVC`; prefer that when possible.
- If a restore creates a separate `volsync-*-dest-dest` PVC instead of restoring into the live claim, copy the restored data into the real app PVC before bringing the app up.
- On this single-node test cluster, Hoarder also needed temporary lower CPU requests to fit alongside the rest of the workload during validation.
- If Longhorn is not ready yet but restore testing must proceed, a temporary `longhorn` StorageClass alias can point at the current OpenEBS hostpath provisioner. Treat that as a migration-only workaround, not the final storage design.
- For Servarr apps, the most reliable restore source was the built-in scheduled backup zip in `/config/Backups/scheduled`, not the older chart-managed VolSync path.
- After restoring Servarr data from the native backup zips, explicit standalone VolSync `ReplicationSource` objects were added so backup uploads resume from the now-correct PVC data.
- On a single-node test cluster with limited vCPU, some control-plane-adjacent workloads needed temporary replica reductions to avoid scheduler deadlock while validating apps:
  - `descheduler` from `2` to `1`
  - `kubelet-csr-approver` from `3` to `1`
  - `cloudnative-pg` operator from `2` to `1`
  - `authelia` from `2` to `1`
- LLDAP could not be restored from the CNPG object-store backups because the required WAL segments were missing from the archive, even when targeting older backup IDs.
- The working LLDAP recovery path was:
  1. bootstrap a fresh CNPG cluster with `initdb`
  2. import the SQL dump from `~/Downloads`
  3. restore `/data/private_key` from the LLDAP data tarball
  4. start LLDAP before starting Authelia
- Authelia's database restore was not the blocker; Authelia started normally once LLDAP was healthy again.
- Longhorn did not install cleanly while `recurring-jobs.yaml` was part of the same kustomization, because those custom resources were applied before the Longhorn CRDs existed.
- For bootstrap, install Longhorn first and apply recurring jobs only after the CRDs are available, or keep recurring jobs out of the initial kustomization.
- Even after real Longhorn is installed, several chart-managed VolSync restores still leave the live app PVC in `Pending` because the PVC uses `dataSource: ReplicationDestination` and waits for `status.latestImage` on the restore object.
- In practice, those apps already have restored data in `volsync-*-dest-dest` PVCs, but the live claim never hydrates automatically; they need either:
  1. a manual copy from the restored destination PVC into a plain live PVC, or
  2. a different restore path that writes straight into the live claim.

## Validation Sequence

1. Verify host prep (`lsblk`, `findmnt /var/lib/longhorn`, `swapon --show`).
2. Verify k3s (`systemctl status k3s`, `kubectl get nodes -o wide`).
3. Verify local cluster shape before Flux (`kubectl get pods -A`).
4. Run `kustomize build clusters/main/kubernetes` locally before applying GitOps changes.
5. Reconcile Flux only after the Talos-specific manifests are either removed or updated.

## Current Blockers

- Longhorn is not the active backing provisioner yet, so stateful restore testing is currently using a temporary `longhorn` StorageClass alias backed by OpenEBS hostpath.
- Some operator and infrastructure Helm releases still need dependency cleanup before the cluster reaches full parity.
- The new cluster is intentionally running in a cautious test posture, with some risky apps suspended during restore validation.
- MetalLB is currently not healthy on the new cluster, so `LoadBalancer` services such as `blocky-dns` and the nginx controllers do not have real LAN IPs yet.
- The current MetalLB failure path is: chart pull denied from `ghcr.io/home-operations/charts-mirror/metallb:0.14.9`, followed by `metallb-config` failing because the MetalLB CRDs never become available.
