# Data architecture

## TL;DR

- **TrueNAS** is the storage floor: ZFS pools, redundant vdevs,
  scheduled snapshots, exports to both Proxmox and Kubernetes.
- **NFS** is the shared-storage path. ReadWriteMany for Kubernetes
  PVs and for Proxmox guests that need shared files.
- **PostgreSQL** on a dedicated VM is the "real database" tier.
  Grafana lives here; future multi-writer workloads can too.
- **SQLite on NFS is not a pattern.** It is a footgun, and the lab
  has the scar to prove it
  ([arr-rollout-restart.md](../lessons-learned/arr-rollout-restart.md)).
- **Backups** happen at multiple layers: TrueNAS replicates pools,
  scheduled config backups capture TrueNAS itself, and application
  state (PostgreSQL, Grafana provisioning) is handled in code where
  possible.

## Deep dive

### Storage layers

Four tiers, each for a specific class of workload:

1. **ZFS datasets on TrueNAS.** The canonical storage pool. Datasets
   split by domain (media, application state, backups). Snapshots
   are scheduled and retained per dataset; the important ones
   replicate off-box.
2. **NFS exports from TrueNAS** to Kubernetes and to Proxmox guests
   that need shared files. ReadWriteMany by nature.
3. **Local block storage** on hypervisor hosts for anything that
   needs low latency and does not need to be shared (database
   files, etcd).
4. **Object-ish** not yet used in anger. If Loki log volume grows,
   S3-compatible object backing (MinIO on TrueNAS) is the
   pre-planned next step.

### NFS: where it fits, where it does not

NFS is the path of least resistance for shared storage, and I use it
for:

- **Media libraries.** Jellyfin reads from large NFS-mounted
  datasets. Writers are controlled (only the *arr apps write into
  the library), so the concurrent-writer problem does not apply.
- **Static application config** for workloads that do not use
  SQLite or otherwise assume single-writer file locking.
- **Vector log paths** on VMs are local; NFS is not in the log
  ingestion hot path.
- **Loki chunks** for now. Future work is S3/MinIO.

NFS is not the path for:

- **SQLite databases.** The combination of SQLite's assumption that
  it is the only writer and NFS's "advisory and not cross-client
  consistent" locking means two pods hitting the same database
  file, even briefly, is undefined behaviour. The *arr apps are the
  canonical case and they live on NFS today for historical reasons;
  the follow-up is to migrate them to RWO block volumes, captured
  in [arr-rollout-restart.md](../lessons-learned/arr-rollout-restart.md).
- **etcd.** Never. etcd expects local disk with sane `fsync`
  semantics. It sits on local storage on the control-plane nodes.
- **PostgreSQL.** Runs on a dedicated VM with local block storage.

### PostgreSQL

The "real database" tier. One VM, dedicated resources, `pg_hba.conf`
wired to accept connections from the Kubernetes node IPs and the
pod CIDR. Today it serves:

- **Grafana.** After the Grafana VM-to-Kubernetes migration, state
  moved from SQLite to PostgreSQL via `pgloader` (4,591 rows; full
  story in [grafana-k8s-migration.md](../lessons-learned/grafana-k8s-migration.md)).
  Moving to PostgreSQL unblocked future HA Grafana deployments.

Future workloads that want a real database point here instead of
adopting SQLite-plus-NFS.

### Media layout

The media stack deserves its own note because it is half the
user-facing lab:

- **Libraries are typed, not mixed.** After a Jellyfin 10.11 upgrade
  made mixed-library edge cases painful, the library is split into
  a Movies library and a TV Shows library, each with the correct
  content type. Captured in
  [jellyfin-library-split.md](../lessons-learned/jellyfin-library-split.md).
- **Dedup is run before index.** Jellyfin is not asked to
  deduplicate; the filesystem is deduplicated at ingest.
- **User data (watch progress, favourites, counts) is the thing
  that matters.** Any library rebuild preserves UserData via the
  10.11 auth header + UserData preservation API trick. Losing this
  data is worse than losing the file index.

### Backup and disaster recovery

Three concentric circles, deliberately.

**Inner: dataset snapshots.** TrueNAS takes regular ZFS snapshots
on the important datasets. Rollback from a snapshot is fast and
cheap, and covers the "oh no I deleted the wrong thing" case
without involving any recovery machinery.

**Middle: application state captured in code.** Grafana dashboards
are provisioned from [`observability-quasarlab`](https://github.com/mithr4ndir/observability-quasarlab).
Kubernetes manifests are in [`k8s-argocd`](https://github.com/mithr4ndir/k8s-argocd).
Host configuration is in [`ansible-quasarlab`](https://github.com/mithr4ndir/ansible-quasarlab).
If a pod's state is lost, re-provisioning is one ArgoCD sync away.

**Outer: config backups.** The `truenas-config-backup` (private)
repo takes scheduled snapshots of the TrueNAS configuration itself,
so a failed TrueNAS upgrade is recoverable without losing dataset
layout, share definitions, or user permissions.

The full DR playbooks live in the private
`quasarlab-disaster-recovery` repo; the shape of that repo is
documented in the public [docs/security/README.md](../docs/security/README.md).

### Failure modes and responses

- **Storage VM fails hard.** Pods requesting NFS PVs go into
  pending. Kubernetes does not kill running pods that have already
  mounted; they run read-only or fail on write. Alertmanager fires
  `NfsStaleMount` and related rules. Recovery is to restore
  TrueNAS (either boot it back up or restore from a config
  snapshot) and reconnect the exports; NFS clients reconnect
  automatically.
- **Dataset corruption.** Rolling back to the most recent
  snapshot is the first move. ZFS scrub runs on a schedule and
  alerts on checksum errors.
- **A writer-on-NFS footgun fires.** Known example: *arr SQLite
  corruption. Stop the writers, restore the DB from the most
  recent backup, then apply the scale-0-then-1 operating rule
  going forward.
- **Object-store need arrives.** Loki backing swaps from NFS to
  MinIO via a Helm values change; no client config change needed.

### Related code

- [`ansible-quasarlab`](https://github.com/mithr4ndir/ansible-quasarlab)
  host-side NFS mount management, TrueNAS configuration where it
  touches the fleet.
- [`k8s-argocd`](https://github.com/mithr4ndir/k8s-argocd)
  NFS subdir provisioner, PVCs, and the PostgreSQL-backed
  `ExternalSecret` wiring.
- [`terraform-quasarlab`](https://github.com/mithr4ndir/terraform-quasarlab)
  the PostgreSQL VM and its provisioning.
- `truenas-config-backup` (private) scheduled TrueNAS config
  capture.
