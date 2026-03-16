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
5. Install Flux.
6. Reconcile the repo after the Talos-only kustomizations are removed from this branch.

## Backup Safety

- Migration-specific backup writer guardrails are recorded in `docs/migration-backup-guardrails.md`.
- Until the new cluster is stable, the branch is configured so it can restore from backups without becoming a backup writer.

## Manual Bootstrap Notes

- Cilium was manually bootstrapped with Helm `v3.19.0`.
- The chart version used was `1.18.6`, matching `clusters/main/kubernetes/kube-system/cilium/app/helm-release.yaml:1`.
- The bootstrap values matched the repo values, except the Kubernetes API endpoint was set to `home-apps.lan.1al.cc:6443`.
- The fresh host was missing `/etc/cni/net.d`, so that directory had to be created before Cilium could write `05-cilium.conflist` and make the node schedulable.

## Validation Sequence

1. Verify host prep (`lsblk`, `findmnt /var/lib/longhorn`, `swapon --show`).
2. Verify k3s (`systemctl status k3s`, `kubectl get nodes -o wide`).
3. Verify local cluster shape before Flux (`kubectl get pods -A`).
4. Run `kustomize build clusters/main/kubernetes` locally before applying GitOps changes.
5. Reconcile Flux only after the Talos-specific manifests are either removed or updated.

## Current Blockers

- Flux is not bootstrapped yet because this migration branch has not been committed and pushed for the new cluster to reconcile from.
- The repo changes in this branch should be committed before connecting the new cluster to GitOps.
