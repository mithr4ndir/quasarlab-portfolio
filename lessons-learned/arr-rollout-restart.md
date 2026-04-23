# Never rollout-restart *arr apps on NFS

## Date

Incident: 2026-03-15. Operating rule in effect ever since.

## Impact

Complete SQLite database corruption (zeroed file header) on a *arr app
after a routine `kubectl rollout restart`. Recovery required restoring
from backup and re-verifying library state.

## Duration

Corruption was near-instant. Recovery and verification took an
evening.

## Category

kubernetes, storage, databases, media stack

## Summary

Rolling restarts of *arr apps (Sonarr, Radarr, Bazarr, and friends)
backed by SQLite on NFS-mounted PVCs corrupt the database. The
corruption is not intermittent or theoretical; in this lab it zeroed
the SQLite header on the first try. The operating rule is: scale the
Deployment to zero, wait, then scale back to one. Never
`kubectl rollout restart` these workloads.

## Background

*arr apps store their state in SQLite. The PVC backing their config
directory is on NFS (ReadWriteMany, served by TrueNAS). This layout
is common in homelabs because NFS is easy and shared storage is
convenient.

SQLite assumes it is the only writer to its database files. It uses
OS-level file locking to coordinate with itself; it does not defend
against two independent processes opening the same database
simultaneously.

NFS advisory locks are implemented, but they are advisory. They are
also not guaranteed to be enforced consistently across clients,
especially during brief overlap windows where two clients both think
they hold a lock.

## What happened

Routine maintenance. Ran `kubectl rollout restart deploy/<arr>` on an
*arr app to pick up a configuration change. Kubernetes, with the
default `RollingUpdate` strategy, created a new pod before terminating
the old one. For a few seconds, two pods were alive. Both mounted the
same NFS-backed config volume. Both opened the SQLite database.

The new pod came up. The old pod was terminated. The database was
destroyed. The file header was zeroed; the database was unreadable.

## Investigation

The failure mode was obvious in hindsight:

- `kubectl rollout restart` respects `spec.strategy`. The default is
  `RollingUpdate` with `maxSurge=25%` and `maxUnavailable=25%`, which
  for a single-replica Deployment rounds up to "create a new pod
  before deleting the old one."
- During the overlap, both pods hold the SQLite database open. NFS
  advisory locking does not stop this; SQLite trusts the lock and
  writes.
- Two writers to a SQLite file are undefined behaviour. The
  observed outcome was a zeroed header, but "any outcome other than
  correct" is on the menu.

## Root cause

Structural. SQLite + NFS + rolling restart is not a supported
combination for any workload. *arr apps happen to be the common case
in homelabs because they default to SQLite and everything else is on
NFS.

## Fix

For any *arr app, the restart path is:

```bash
kubectl scale deploy/<name> -n media --replicas=0
# wait for the pod to fully terminate
kubectl scale deploy/<name> -n media --replicas=1
```

The `sleep` between the two scales is a safety belt; what actually
matters is that the old pod is gone before the new one arrives. A
`kubectl wait --for=delete pod -l app=<name>` is the rigorous version.

Alternatives considered:

- **`strategy: Recreate`** on the Deployment. Works, because it forces
  delete-before-create. Downside: it can fight ArgoCD sync behaviour
  in some configurations, and it is not the cluster-wide default. The
  scale-0-then-1 pattern is more portable and does not require a
  manifest change.
- **Move off NFS.** The right long-term answer. An RWO PVC backed by
  local-path or a dedicated block storage class removes the whole
  failure mode. Listed as a follow-up.

Alerting and prevention:

- The operating rule is documented in team memory and referenced in
  any runbook that touches media. Do not expect people to rediscover
  it.
- The *arr apps are tagged in inventory so their restart procedure
  is distinct from "generic Kubernetes Deployment restart."

## Takeaways

- Rolling restarts are a default that does not survive every storage
  backend. Know your backend before you trust the default.
- SQLite + NFS + any overlap equals corruption. There is no "rare
  case" here; if two writers can be there at once, they eventually
  will be.
- `strategy: Recreate` is a reasonable default for any workload
  where the application assumes single-writer semantics. Stateless
  is the happy path; many real apps are not stateless.
- Operating rules need to be discoverable in the places people
  actually look (runbooks, inventory tags, memory for agents).
  A rule nobody finds is not a rule.

## Follow-ups

- [ ] Move *arr SQLite databases off NFS entirely, onto RWO PVCs
      (local-path or a dedicated block-storage class).
- [ ] Consider `strategy: Recreate` on the Deployments as a belt
      while the NFS move is pending.
- [ ] Capture the `wait --for=delete` variant of the restart
      procedure in the private runbook repo.
