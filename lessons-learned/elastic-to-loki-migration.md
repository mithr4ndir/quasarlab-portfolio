# Elastic Stack to Loki + Wazuh migration

## Date

TODO

## Impact

Log platform overhaul. Every service that produced logs had to re-route.
Every dashboard that queried Elastic had to move or be rebuilt.

## Duration

TODO

## Category

observability, logging, migration, cost

## Summary

The lab moved off the Elastic Stack (Elasticsearch + Logstash/Beats +
Kibana) to Loki + Promtail for general log aggregation and retained
Wazuh for SIEM, file integrity monitoring, and compliance-flavoured
workloads. This is the flagship retrospective of the repo; the points
below are the scaffolding for a proper writeup.

## Background

The following prompts define what this section needs to answer. They
are intentionally left as TODO so the writeup reflects the actual
history rather than a plausible-sounding reconstruction.

- TODO: How was the ELK/Elastic Stack originally deployed? Standalone
  VMs, Kubernetes-hosted ECK, which version? What was it ingesting
  (platform logs, syslog from network gear, application logs, Wazuh
  alerts)?
- TODO: What was the retention policy and roughly what was the index
  footprint (orders of magnitude, not exact bytes)?

## What happened

TODO describe the decision moment. Was it a single trigger (cost,
licensing, an incident) or accumulated pressure?

## Investigation

TODO describe the alternatives considered (Elastic stays, Loki, Grafana
Cloud, OpenSearch) and the comparison axes used.

## Root cause (of the migration decision)

Not a root cause in the incident sense, but the set of forces that
pushed the lab off Elastic. The writeup needs to answer:

- TODO: Resource cost. JVM heap pressure, node sizing, disk IO profile.
- TODO: Licensing. Post-2021 SSPL change and how it did or did not
  affect the homelab use case.
- TODO: Operational complexity. ILM tuning, cluster health, shard math,
  upgrade pain.
- TODO: Storage growth trajectory.
- TODO: UX. Grafana as a single pane of glass for metrics and logs vs
  context-switching to Kibana.

## Fix

Not a fix, a replacement. The writeup needs to answer:

- TODO: What replaced what? Loki took over general log aggregation.
  Wazuh retained for SIEM/FIM/compliance. Confirm whether anything
  still lives in Elastic or whether it is fully decommissioned.
- TODO: Migration approach. Parallel run duration, cutover date, how
  clients (apps, agents, syslog sources) were repointed.
- TODO: Historical log retention. Kept old indices online? Snapshot to
  object storage? Dropped?
- TODO: Decommissioning checklist results per the project standard:
  - Prometheus static targets removed for the old Elastic nodes.
  - Ansible inventory entries removed.
  - Terraform state cleaned.
  - Alertmanager routes and PrometheusRules specific to Elastic
    removed or silenced.
  - Grafana dashboards and datasources updated.
  - DNS, NPM upstream, and ingress entries cleaned.
  - Post-decommission: no stray alerts, no stray scrape targets.

## Takeaways

- TODO: When is Loki the right tool? Label cardinality constraints,
  full-text search expectations, query patterns.
- TODO: When is Elastic still the right tool? Full-text search depth,
  aggregation-heavy analytics, specific Kibana features.
- TODO: What was lost or traded away? Kibana features that did not port,
  dashboards that had to be rebuilt, search patterns that had to change.
- TODO: Decommissioning is its own skill. Removing a cluster is not
  done when the VMs are off; it is done when the monitoring,
  automation, and inventory all stop referring to it.

## Follow-ups

- [ ] Fill in every TODO above from real history before publishing this
      file widely.
- [ ] Link to the specific PRs in `observability-quasarlab`,
      `k8s-argocd`, and `ansible-quasarlab` that executed the cutover.
- [ ] Add a before/after screenshot pair showing the Grafana logs
      experience vs the old Kibana experience (sanitised).
