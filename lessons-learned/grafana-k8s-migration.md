# Grafana VM to Kubernetes migration

## Date

Completed 2026-04-07.

## Impact

Grafana moved off a standalone VM and into the Kubernetes cluster as a
proper Helm-managed release inside `kube-prometheus-stack`. The old VM
was destroyed cleanly. One follow-up remained: an Angular-based panel
plugin in CrashLoop on Grafana 12.x.

## Duration

Planned migration, not an incident. Cutover was done in a maintenance
window with the usual "Grafana briefly unavailable" downtime.

## Category

observability, migration, database, decommissioning

## Summary

Grafana 12.4.1 deployed via `kube-prometheus-stack`, backed by a
dedicated PostgreSQL instance (pgloader-migrated from the old SQLite
database preserving all 4,591 rows), exposed via a MetalLB LoadBalancer,
users and dashboards reconstructed, and the original VM decommissioned
with a full checklist rather than just "delete the VM."

## Background

The pre-migration Grafana was a single VM running `grafana-server.service`
from the Debian package. State lived in SQLite. Dashboards were
provisioned from file via an Ansible role that synced the
`observability-quasarlab` repo to the VM.

This worked, and it would have kept working. The reasons to move:

- Single-host unplanned downtime equalled Grafana downtime. No
  horizontal scaling, no rolling restart.
- VM-based observability tools were inconsistent with the rest of the
  observability stack, which had already consolidated into Kubernetes
  (Prometheus, Alertmanager, Loki, Vector, Wazuh notwithstanding as a
  deliberate separate VM).
- Dashboard provisioning from a VM required an out-of-band Ansible
  playbook. Moving into Kubernetes let provisioning flow through
  ArgoCD like every other workload.

## What happened

The migration plan had four real moving parts:

1. Stand up the new Grafana in Kubernetes with an empty state store.
2. Migrate state (users, orgs, dashboards, alerts) from SQLite to
   PostgreSQL.
3. Cut clients over to the new address.
4. Decommission the old VM cleanly.

## Investigation / design decisions

**State store**: Grafana supports SQLite, MySQL, and PostgreSQL.
SQLite on a ReadWriteMany volume would have reproduced the *arr
NFS + SQLite footgun (see
[arr-rollout-restart.md](arr-rollout-restart.md)). PostgreSQL on a
dedicated VM won: proper concurrent writer semantics, Grafana
horizontal scaling becomes possible, backup story is standard `pg_dump`.

**Migration tool**: `pgloader` for SQLite to PostgreSQL. One command,
whole database, type conversions handled. 4,591 rows migrated cleanly.

**Ingress**: a MetalLB LoadBalancer IP for Grafana, sitting in the
same pool as Prometheus, Alertmanager, and Loki. Nginx Proxy Manager
upstream updated to point at the new IP.

**Secrets**: Grafana DB credentials come from 1Password via External
Secrets Operator. No plaintext anywhere in `values.yaml`.

## Root cause

Not applicable. Planned migration.

## Fix / implementation

- Grafana 12.4.1 deployed via `kube-prometheus-stack` Helm chart with
  PostgreSQL backend.
- Dedicated PostgreSQL database on a standalone VM. `pg_hba.conf`
  configured to accept connections from the Kubernetes node addresses
  and the pod CIDR.
- SQLite to PostgreSQL migration via `pgloader`; 4,591 rows migrated,
  verified against the source DB row count.
- Six users recreated via the Grafana Admin API, passwords set to a
  temporary value with a forced reset on first login.
- 36 dashboards confirmed loading (17 custom provisioned via ConfigMap
  plus 19 built-in from `kube-prometheus-stack`).
- ExternalSecret wired up for Grafana DB credentials and for the
  Wazuh-related shared credentials that Grafana needs for its Wazuh
  panels.
- MetalLB LoadBalancer from the auto-assigned observability pool.

Decommissioning checklist (per the standing decommission rule):

- Prometheus static target for the old VM removed from the file_sd
  ConfigMap.
- Homepage dashboard updated to the new LoadBalancer IP.
- Ansible `host_vars/grafana/` directory deleted.
- Dynamic Proxmox inventory auto-cleaned once the VM was destroyed.
- PVE exporter auto-drops the old VMID.
- `kube-prometheus-stack` values: any reference to the old VM IP
  removed.
- NPM upstream updated from the old VM address to the new
  LoadBalancer.

## Takeaways

- "Migrate to Kubernetes" is three migrations hiding in a trench coat:
  workload, state, and the decommissioning of what you just replaced.
  The last one is the one most people skip.
- SQLite is a great single-writer database. The moment a migration
  invites the possibility of two writers, pick PostgreSQL.
- `pgloader` is the right tool for "move a SQLite file to PostgreSQL
  in one step." It is worth the `apt install`.
- Angular-based Grafana panels are a migration hazard on modern
  Grafana. Check the plugin catalogue for deprecation notes before
  bumping versions.
- A LoadBalancer IP in Kubernetes is cheaper and more portable than an
  NPM upstream pointing at a VM IP. The NPM upstream is still there,
  but it has one less indirection behind it.

## Follow-ups

- [ ] Replace or remove the Angular-based panel plugin that is in
      CrashLoop on 12.x.
- [ ] Rotate the temporary admin password properly and store the final
      value in 1Password.
- [ ] Document the PostgreSQL backup cadence (`pg_dump` schedule) in
      the storage docs.
- [ ] Consider running two Grafana replicas now that PostgreSQL
      supports it; currently running one for simplicity.
