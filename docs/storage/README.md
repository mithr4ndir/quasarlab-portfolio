# Storage

Storage layer: TrueNAS, NFS exports to Kubernetes, and PostgreSQL.

## What is here

- TrueNAS pool and dataset conventions (described, not enumerated).
- NFS export patterns and how Kubernetes PVs consume them.
- PostgreSQL usage per workload and backup strategy.

## Conventions

- One dataset per logical domain (media, backups, app data).
- NFS is only used where ReadWriteMany is actually required. SQLite + NFS
  is a known footgun; see `lessons-learned/arr-rollout-restart.md`.

TODO fill in retention, snapshot schedule, and DR drill cadence.
