# Proxmox HA fence, nine hours of silence

## Date

Incident: 2026-04-13. Multi-day remediation session: 2026-04-14 through
2026-04-18.

## Impact

A Proxmox node was HA-fenced. VMs that should have restarted on the
surviving node did not, and stayed down for roughly nine hours. During
that window, not one alert fired about the VM downtime.

## Duration

Nine hours of silent VM outage. Remediation ran across four days and
touched two repos and 21 merged pull requests (10 in `ansible-quasarlab`,
11 in `k8s-argocd`).

## Category

virtualization, high-availability, alerting, incident response, supply
chain, Kubernetes upgrades

## Summary

A rare boot-time race between a VM and its TrueNAS NFS dependency meant
that when HA fenced the host, the restarted VM did not come back. The
alert path had a separate latent gap: VM-level alerts that should have
caught "VM is down on a guest that is supposed to be running" did not
fire cleanly. The remediation went well past "fix the fence" and turned
into a multi-day pass across boot dependencies, alerting, 1Password
rate-limit hardening, and a set of long-overdue infrastructure upgrades.

## Background

Two Proxmox nodes run as an HA pair. Some VMs depend on TrueNAS NFS
mounts being available at boot. The HA group in Proxmox handles node
fencing; `onboot=1` on guests is supposed to handle restart.

## What happened

HA fenced one of the Proxmox nodes. The surviving node tried to
restart the VMs with `onboot=1`. One of them (the FreeNAS mount
provider) came back in a state where its NFS-dependent guests could
not acquire their mounts during boot. The hookscript that was supposed
to handle this raced with the NFS availability and failed silently,
because the `failed_when` expression did not handle a null condition
correctly.

Meanwhile, the VM-down alert for the affected guests did not fire. The
PVE exporter, Prometheus rules, Alertmanager routes, and the VM-level
`onboot=1` filter all behaved individually correctly; the emergent
behaviour across the chain missed the state.

Nine hours later, I noticed that services were down.

## Investigation

The incident ran down three simultaneous threads:

1. **Why didn't the VMs come back?** A probe race. The `freenas-proxmox`
   boot order waited on TrueNAS, but the wait step did not correctly
   distinguish "API up but auth-required" (healthy) from "API down"
   (still waiting). It treated 401/403 as failure and timed out.
2. **Why didn't we get paged?** The `PveGuestDown` alert had an
   `onboot=1` filter that used a non-anchored regex, and a guest tag
   mismatch meant the alert's matcher did not line up with the guest
   that went down. The alert did not fire for the specific VM shape
   involved.
3. **Why was the hookscript quiet?** A `failed_when: result.rc == X`
   expression compared to a null value without null-safety. The task
   "succeeded" by not failing, which looks identical to "succeeded
   because everything worked."

Each thread turned up its own ecosystem of adjacent issues. By the
time the session wrapped, the remediation had grown to cover etcd,
1Password, and Kubernetes upgrades as well.

## Root cause

Cascading small defects, none individually dramatic, which together
took both the reliability and the observability of the HA restart
path to zero:

- `wait-truenas-api` probe accepted the wrong response codes as "not
  yet ready."
- Hookscript `failed_when` was not null-safe.
- `PveGuestDown` regex was not anchored and the tag filter did not
  match the guest's real tags.
- Legacy `fancontrol` service was active with an empty config, making
  the boot order unnecessarily fragile. BIOS handled fans on these
  nodes anyway.

## Fix

Shipped in `ansible-quasarlab` (PRs #98-#107):

- **freenas-proxmox boot race**: `wait-truenas-api` drop-in that probes
  the API and accepts 2xx plus 401/403 as healthy (auth just means the
  service is up), rejects 5xx, with `TimeoutStartSec=infinity` and a
  300-second default wait, tasks tagged for targeted reruns.
- **fancontrol**: disabled, empty config removed. BIOS handles fans.
- **SSH policy**: switched to `accept-new` so first-boot host keys do
  not hang reboot automation.
- **Reboot metric timer**: added so post-reboot validation has a
  Prometheus signal to lean on.
- **Hookscript null-safe fix**: `failed_when` now handles null
  correctly.
- **1Password kill switch**: an Ansible-side protection that auto-trips
  on rate-limit errors with a 24-hour TTL and exposes a Prometheus
  metric. Any Ansible playbook that hits the kill switch fails fast
  instead of extending the rate-limit window.
- **1Password secret caching**: 12-hour TTL file cache, playbooks read
  through environment-variable lookups with an assert-fail-fast check.
- **etcd weekly defrag timer**: `k8s_maintenance` role runs a Sunday
  04:00 defrag.

Shipped in `k8s-argocd` (PRs #115, #117-#122, #129-#132):

- **Watchdog to Healthchecks.io**: the dead-man's-switch now lives
  outside the cluster. Prometheus pings, Healthchecks.io raises the
  alarm if the pings stop.
- **Discord alert proxy HA**: two replicas, baked immutable image,
  Dependabot for dependency bumps, Trivy image scan in CI.
- **PveGuestDown**: tag filter now uses an anchored regex, runbook
  link added to the alert annotation.
- **Dask alert rules**: `job` label and `sum by state` aggregation
  fixed.
- **Homepage widget fixes**: NZBGet port, Alertmanager external service
  address.
- **ESO refresh**: raised to 24 hours to stay inside 1Password
  rate-limit windows.
- **etcd db-size alerts**: added, even though etcd metrics still need
  to be fully exposed to the Prometheus scrape path (follow-up).
- **Grafana reloader annotation**: so ConfigMap changes flow into the
  Grafana pod without manual intervention.

Manual operations in the same window:

- QDevice NSS database restored on the primary Proxmox node. Cluster
  quorum restored to three votes.
- Both Proxmox hosts upgraded to 8.4.18.
- Both Proxmox hosts rebooted. All VMs auto-started cleanly; the
  FreeNAS fix was validated in anger.
- etcd compacted and defragmented (about 124 MB down to 46 MB).
- Kubernetes rolling upgrade: all three nodes upgraded to v1.33.11
  plus kernel 6.8.0-110.
- External Secrets Operator token rotated to a dedicated service
  account, scaled to zero during rate-limit cooldown, scaled back
  once the window cleared.
- Authentik monitoring API token regenerated, Kubernetes Secret
  patched in place.

## Takeaways

- "HA works" is a claim that requires a real node failure to verify.
  Rehearse node fencing on a schedule; do not wait for production to
  rehearse for you.
- Boot-time dependencies are invisible until they fail. Probe
  scripts need clear definitions of "ready" that match the real-world
  failure modes (auth errors are ready; 5xx is not ready).
- Alert regexes need anchoring. Tag-filter regexes need to match the
  real tags on the real guests; write the unit test.
- Silent success is a bug. `failed_when` against null is a silent
  success. Any task that can succeed without doing its job is a
  reliability hole.
- When an incident reveals an adjacent weakness, it is cheaper to fix
  it in the same wave than to open a ticket. In this case the
  remediation expanded to 1Password hardening, etcd defrag
  automation, Kubernetes upgrades, and a real external dead-man's-
  switch. All of them were overdue; the outage made them urgent.

## Follow-ups

- [x] Rotate 1Password tokens to scoped service accounts.
- [x] Kubernetes cluster on current minor.
- [x] etcd defrag automation.
- [x] External dead-man's-switch.
- [ ] Physical: one node still has a missing NVMe that needs hands-on.
- [ ] etcd Prometheus metrics are still not fully exposed (binds to
      127.0.0.1, `kubeEtcd` disabled). Alerts exist; the metric
      source needs a proper Service.
- [ ] Schedule a quarterly "kill a node" exercise to keep HA claims
      honest.
