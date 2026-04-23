# Elastic Stack to Loki + Vector + Wazuh migration

## Date

Completed 2026-03-15.

## Impact

Full log-platform swap. Every service that produced logs changed shipping
path. Every dashboard that queried Elastic was either migrated to Loki
(LogQL) or rebuilt. Freed meaningful RAM (tens of GBs) back to the
hypervisor.

## Duration

Multi-phase rollout over a parallel-run window. Cutover on 2026-03-15.

## Category

observability, logging, migration, cost, decommissioning

## Summary

The lab moved off the Elastic Stack (Elasticsearch + Logstash + Kibana,
all-in-one on a single VM) to Loki + Vector for general log aggregation,
and retained Wazuh for SIEM, file integrity monitoring, and
compliance-flavoured workloads. Elastic is fully decommissioned. This is
the flagship retrospective of the repo.

## Background

Original deployment: a single-node ELK setup, one VM running Logstash,
Elasticsearch, and Kibana together. The choice was made because it was
what I had used professionally at a previous employer: well understood,
fast to stand up, known operational shape.

It was ingesting:

- Platform logs from Kubernetes workloads via Filebeat.
- System logs (journald, /var/log) from all the non-Kubernetes VMs, also
  via Filebeat.

It worked. It was also, by honest measurement, overkill for the amount
of log volume a homelab actually produces.

## What happened

No single triggering incident. Pressure accumulated:

- JVM memory appetite. A single-node Elasticsearch with enough heap to be
  stable took more RAM than the rest of the observability stack
  combined.
- Operational complexity. Index lifecycle management, shard math, cluster
  health colour codes, periodic yellow-state investigations. All of it
  legitimate for a production tenant serving an org; pure drag for a
  personal lab.
- UX. Kibana and Grafana were two separate panes of glass. Context
  switching between them for a single investigation (metric spike, then
  logs) was a small friction every single time.
- Growth. Disk growth was a predictable monthly chore. The ratio of
  "useful logs I actually query" to "logs I am paying storage and RAM
  for" was wrong.

None of these were red alarms individually. Together they made
"continue with Elastic" the worse option.

## Investigation

Alternatives considered:

- Stay on Elastic, right-size. Rejected: the RAM floor was the problem,
  and right-sizing below that floor meant cluster instability.
- Loki + agent. Low memory footprint, label-based model fits a homelab's
  query patterns (drill in by host, namespace, app), Grafana-native.
- Move to a hosted offering. Rejected on principle: the whole point of
  the lab is running the stack, and no data should leave the LAN by
  default.

Loki won on resource profile, on unified UX with Grafana, and on "the
questions I actually ask of logs fit LogQL."

Separate decision: Wazuh stays. Wazuh does SIEM, FIM, and compliance
rules; Loki does not. Replacing Wazuh with "Loki plus rules I write
myself" would have been rebuilding a SIEM badly.

Promtail was briefly considered. Vector won for three reasons: one tool
covers both the VM agent role and the Kubernetes DaemonSet role; VRL
transforms do real work (label shaping, noise drops) at the edge
instead of pushing the mess into Loki's cardinality; clearer
observability story on the shipper itself.

## The migration decision

- Resource cost: JVM heap pressure was the single biggest factor.
- Licensing: post-2021 SSPL did not directly affect a self-hosted lab,
  but it signalled that future ecosystem direction was not going to
  ease the homelab case.
- Operational complexity: ILM, shard sizing, and cluster health
  investigations were not paying for themselves at this scale.
- UX: one Grafana for metrics and logs is materially better than
  metrics-in-Grafana, logs-in-Kibana.
- Storage growth: predictable and boring, but a standing monthly tax.

## What replaced what

Loki + Vector took over everything Elastic was doing for general logs:

- **Loki** SingleBinary deployment in Kubernetes, backed by NFS, 30-day
  retention.
- **Vector DaemonSet** for Kubernetes container logs, with noisy
  namespaces (kube-system, monitoring) excluded so Loki cardinality
  stays sane.
- **Vector agent** on every non-Kubernetes VM, shipping journald and
  `/var/log/**/*.log`.
- **Grafana dashboards**: VM Log Explorer and Kubernetes Log Explorer,
  both with host/namespace/app filters. LogQL
  `| json | line_format` for clean message rendering. NVIDIA GPU noise
  filtered at the agent, not the query. `app` label extracted from log
  paths so dashboards remain snappy.

Wazuh took the SIEM and FIM lane:

- All-in-one Wazuh (Manager + Indexer + Dashboard) on its own VM.
- Wazuh agents on every Linux host, grouped by role (linux, kubernetes,
  proxmox).
- Filebeat 7.10.2 on the Wazuh manager (8.x is incompatible with the
  Wazuh Indexer's OpenSearch version; one of the migration gotchas).
- TLS certificates generated via `wazuh-certs-tool`.
- Admin password sourced from 1Password.

Grafana itself was still on a VM during the Elastic cutover. It was
later migrated into Kubernetes (see
[grafana-k8s-migration.md](grafana-k8s-migration.md)).

## Migration approach

Parallel run: Loki + Vector stood up alongside the Elastic stack so
dashboards could be rebuilt and validated before anything was turned
off. Clients (Filebeat consumers) were repointed by removing Filebeat
and deploying Vector, host by host.

Historical retention: old Elastic indices were not preserved. The data
that mattered (dashboards, alert rules, SIEM state) lived elsewhere.
Pure log history was dropped cleanly; the disk space was the point.

## Decommissioning

The Elastic decommission followed the standing checklist rather than
"destroy the VM and hope":

- Filebeat removed from every VM via package removal.
- Filebeat DaemonSet and its ArgoCD Application removed from Kubernetes.
- Elasticsearch ExternalSecret removed.
- Elasticsearch VM destroyed, freeing a large RAM allocation back to
  the hypervisor.
- Elastic removed from Ansible inventory and Prometheus static targets.
- Old `elasticsearch-logs.json` Grafana dashboard deleted.
- Duplicate Grafana alert rules (which had been shadowing Prometheus
  rules) deleted.
- Stale Elasticsearch datasources cleaned from Grafana.

Post-decommission checks: no stray scrape targets, no alerts firing on
a ghost host, no inventory entries referring to the old VM.

## Takeaways

- Pick the simplest tool that fits the query pattern you actually have.
  Elastic is a fine tool; it was the wrong tool for this workload.
- The biggest cost of a log platform is rarely storage. It is operator
  attention: JVM tuning, ILM, cluster colour codes, upgrade cycles.
- Loki + Vector is strong for "drill by host or namespace or app". If
  the question is "find this exact phrase across 90 days," Elastic or
  OpenSearch are still the right answer.
- Wazuh and Loki are complements, not substitutes. SIEM is its own
  job.
- Decommissioning is a skill. The VM going away is the first step, not
  the last. Monitoring, inventory, and automation all have to let go
  too.
- Infrastructure choices inherited from prior jobs deserve a fresh
  evaluation against the current environment. What fit a large org may
  not fit a homelab, and that is a design decision in its own right.

## Follow-ups

- [ ] Add a before/after screenshot pair (sanitised) showing the Grafana
      logs experience vs the old Kibana experience.
- [ ] Link to the specific PRs in `observability-quasarlab`,
      `k8s-argocd`, and `ansible-quasarlab` that executed the cutover
      once the underlying repos are published or referenced publicly.
- [ ] Capture the Vector configuration patterns (VRL noise drops, label
      extraction) as a standalone runbook entry.
