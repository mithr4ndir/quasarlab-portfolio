# Never rollout-restart *arr apps on NFS

## Date

TODO

## Impact

SQLite database corruption on *arr apps (Sonarr, Radarr, and friends)
when storage is NFS and the restart path is `kubectl rollout restart`.

## Duration

TODO

## Category

kubernetes, storage, media

## Summary

Rolling restarts of *arr apps backed by SQLite on NFS cause database
corruption. The correct pattern is to scale the Deployment to 0, wait
for the pod to terminate cleanly, then scale back to 1.

## Background

*arr apps use SQLite. SQLite and NFS do not agree on locking semantics
in the presence of two concurrent writers, even briefly.

## What happened

TODO describe the first time this bit: what broke, what the DB looked
like afterwards, how long it took to restore.

## Investigation

TODO

## Root cause

`kubectl rollout restart` creates a new pod before terminating the old
one (by default). For a few seconds, two pods hold the SQLite DB open
on the same NFS share. NFS advisory locks are not honoured reliably
across clients, so both writers corrupt the file.

## Fix

For *arr apps specifically:

- `kubectl scale deploy/<arr> --replicas=0`
- Wait for the pod to terminate.
- `kubectl scale deploy/<arr> --replicas=1`

A Deployment strategy of `Recreate` also works but has the downside of
fighting with ArgoCD sync behaviour in some setups.

## Takeaways

- SQLite + NFS + overlap = corruption. Always.
- "Rolling restart" is a default that does not survive every storage
  backend. Know your backend.

## Follow-ups

- [ ] Document the scale-0-then-1 pattern in the private runbook repo.
- [ ] Consider moving *arr SQLite databases off NFS entirely, onto
      local-path or a dedicated PVC with RWO.
