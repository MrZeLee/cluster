download the file https://raw.githubusercontent.com/longhorn/longhorn/v1.6.2/deploy/prerequisite/longhorn-nfs-installation.yaml

check the version

## Disk tags (manual, required on every new node/cluster)

The default `longhorn` storage class only schedules on disks tagged `ssd`
(`persistence.defaultDiskSelector` in `values.yaml`). This keeps chatty
volumes (databases, app configs) off the spinning-HDD pool. Longhorn does
NOT auto-detect disk types — tags are set by hand, and a disk with no tag
receives no default-class volumes at all.

Convention — every disk carries exactly one *tier* tag, plus optional
*qualifier* tags. `diskSelector` matching is AND-only (a volume matches a
disk only if the disk has ALL the volume's tags, no OR), which is why all
fast disks share the `ssd` tier tag instead of each having only its own kind:

- `ssd` (tier) — every fast disk (NVMe/SSD/default `/var/lib/longhorn`
  disks). Required for the default storage class to use the disk.
- `hdd` (tier) — spinning disks. Only the selector-free bulk classes
  (`longhorn-jellyfin-storage`, `longhorn-owncloud`) may land here, as their
  extra replica; `longhorn-minio-storage` selects `ssd` (MinIO scans/writes
  constantly). Currently just n5pro's
  `longhorn-hdd-ext4` disk: a 20 TB ext4-formatted zvol on the spinning-disk
  raidz pool, mounted at `/mnt/longhorn-hdd-ext4`. (Don't trust
  auto-detection for it: zvols report `rotational=0`, i.e. look like SSDs.)
- `encrypted` (tier) — the ZFS-encrypted dataset, used only by the
  `longhorn-encrypted` storage class (see
  `longhorn-encrypted-storage-class.yaml`). Do not also tag it `ssd`.
- qualifiers (optional, purely descriptive until a storage class selects
  them): `nvme` on NVMe disks, `raid` on the server raid0 array, `raidz` on
  the n5pro raidz pool. E.g. n5pro's boot disk is `nvme,ssd` — a future
  NVMe-only class would use `diskSelector: "nvme"`.

Current layout: n5pro boot `nvme,ssd`; n5pro HDD pool `hdd,raidz`;
raspb3-6 + minipc + server boot `ssd`; server raid0 `raid,ssd`.

How to tag — Longhorn UI: **Node → edit node and disks → Add Disk Tag**
(e.g. `ssd`) → Save. Or with kubectl (replaces the disk's whole tag list):

```sh
kubectl get nodes.longhorn.io -n longhorn-system \
  -o custom-columns='NODE:.metadata.name,DISKS:.spec.disks'   # find disk names
kubectl patch nodes.longhorn.io <node> -n longhorn-system --type=merge \
  -p '{"spec":{"disks":{"<disk-name>":{"tags":["ssd"]}}}}'
```

When adding a node (or standing up a new cluster): tag its disks before
expecting default-class volumes to schedule there — until the `ssd` tag is
set, new PVCs may stay unschedulable. Tag HDDs `hdd` first so nothing fast
lands on them by accident.
