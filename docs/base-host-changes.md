# Base Host Changes

This file records the manual base-host changes made for the current Ubuntu + k3s node so they can be turned into automation later.

## Host

- Hostname: `home-apps.lan.1al.cc` / `home-apps`
- OS: `Ubuntu 24.04.4 LTS`
- Kernel: `6.8.0-106-generic`
- Current IP: `192.168.1.28/23`
- RAM: `24 GiB`
- Disk layout at first inspection:
  - `sda`: `64 GiB` OS disk
  - `sdb`: `256 GiB` unused disk intended for Longhorn

## Changes Applied

### 2026-03-15 - Base package prep

Installed packages:

- `qemu-guest-agent`
- `nfs-common`

Already present before changes:

- `open-iscsi`

Command used:

```bash
sudo apt-get update
sudo apt-get install -y qemu-guest-agent nfs-common
sudo systemctl enable --now qemu-guest-agent iscsid open-iscsi
```

Result:

- `qemu-guest-agent` installed and enabled
- `iscsid` enabled
- `open-iscsi` enabled
- NFS client tools installed for existing NFS-backed workloads

### 2026-03-15 - Disable swap for k3s

Commands used:

```bash
sudo swapoff -a
sudo cp /etc/fstab /etc/fstab.bak-opencode
```

Then `/etc/fstab` was updated to comment out the swap entry:

```fstab
# /swap.img    none    swap    sw    0    0  # disabled for k3s
```

Result:

- Runtime swap disabled
- Persistent swap entry disabled in `/etc/fstab`
- Backup of the original file saved at `/etc/fstab.bak-opencode`

### 2026-03-15 - Expand root filesystem

Command used:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

Result:

- Root logical volume expanded from about `30.5 GiB` to the full `60.9 GiB`
- No free space remains in `ubuntu-vg`

### 2026-03-15 - Prepare dedicated Longhorn disk

Commands used:

```bash
sudo parted -s /dev/sdb mklabel gpt mkpart primary ext4 0% 100%
sudo mkfs.ext4 -F -L longhorn /dev/sdb1
sudo mkdir -p /var/lib/longhorn
sudo blkid -s UUID -o value /dev/sdb1
```

Then `/etc/fstab` was updated with a UUID-based mount:

```fstab
UUID=19c56b5d-a169-4007-9ef0-1bbe5559e941 /var/lib/longhorn ext4 defaults,noatime 0 2
```

Result:

- `/dev/sdb1` created as a dedicated `ext4` filesystem for Longhorn
- Filesystem label set to `longhorn`
- Mounted at `/var/lib/longhorn`
- Created the mount point with `mkdir -p /var/lib/longhorn`

### 2026-03-15 - Apply NFS client defaults and safe kernel tuning

Created `/etc/nfsmount.conf` with:

```ini
[ NFSMount_Global_Options ]
nfsvers=4.2
hard=True
nconnect=16
noatime=True
```

Created `/etc/sysctl.d/90-k8s-migration.conf` with:

```sysctl
fs.inotify.max_queued_events = 65536
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 524288
```

Result:

- NFS mounts on the Ubuntu host now follow the intended global client defaults for the cluster
- Inotify limits were raised to safer values for container-heavy workloads

### 2026-03-15 - Create CNI config directory for Cilium

Command used:

```bash
sudo mkdir -p /etc/cni/net.d
```

Result:

- Created the host directory Cilium needs to write `05-cilium.conflist`
- This was required because the fresh Ubuntu install did not have `/etc/cni/net.d`

### 2026-03-15 - Install single-node k3s

Created `/etc/rancher/k3s/config.yaml` and installed:

```bash
curl -sfL https://get.k3s.io | sudo INSTALL_K3S_VERSION='v1.35.2+k3s1' sh -s - server
```

Key config choices:

- embedded etcd via `cluster-init: true`
- existing cluster CIDRs preserved:
  - pod CIDR: `172.16.0.0/16`
  - service CIDR: `172.17.0.0/16`
- disabled bundled components:
  - `traefik`
  - `servicelb`
  - `metrics-server`
  - `local-storage`
- disabled flannel, kube-proxy, and the built-in network policy controller for Cilium
- set `max-pods=250`
- added TLS SANs for:
  - `home-apps`
  - `home-apps.lan.1al.cc`
  - `192.168.1.28`
- the old cluster VIP `192.168.0.10` was not included in the initial TLS SAN set

Follow-up fix applied:

- Removed unsupported kubelet flags:
  - `shutdown-grace-period=15s`
  - `shutdown-grace-period-critical-pods=10s`

Result:

- `k3s` installed and running on `home-apps`
- Node registered as `home-apps`
- Cluster is currently waiting on Cilium, so the node is expected to remain `NotReady` until CNI bootstrap finishes
- Any kubeconfig that still points at `https://192.168.0.10:6443` will need either a VIP route plus matching server certificate SAN or a temporary rewrite to `home-apps.lan.1al.cc` / `192.168.1.28`

### 2026-03-22 - Disable multipathd for Longhorn

Commands used:

```bash
sudo systemctl disable --now multipathd.service multipathd.socket
sudo systemctl mask multipathd.service multipathd.socket
sudo multipath -F
```

Result:

- `multipathd` no longer claims Longhorn block devices as `/dev/mapper/mpath*`
- Prevents kubelet / Longhorn mount failures that show up as `already mounted or mount point busy`
- This host does not need device-mapper multipath, so disabling it is the simplest safe default

## Current Validation Notes

- `qemu-guest-agent`, `open-iscsi`, `iscsid`, and `k3s` are active
- Swap remains disabled
- Root filesystem is now `60 GiB`
- `/var/lib/longhorn` is mounted from `/dev/sdb1`
- `/etc/cni/net.d` exists for Cilium-managed CNI config
- Host-side base prep is complete for the current single-node target

## Pending Base Changes

These are planned but not yet applied:

- No additional base-host changes are currently pending
