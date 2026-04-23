# Observability architecture

## TL;DR

- **Metrics** Prometheus (kube-prometheus-stack) scraping cluster
  workloads, every VM via node_exporter, Proxmox via pve-exporter,
  and TrueNAS.
- **Logs** Loki, with Vector shipping from two places: a DaemonSet in
  Kubernetes and a host agent on every non-K8s VM.
- **SIEM** Wazuh runs beside the metrics/logs stack. Different
  audience, different UI, shared hosts.
- **Alerts** Alertmanager is the one router. A custom in-cluster
  Discord proxy themes notifications by severity. Grafana alerting
  is intentionally disabled.
- **Dead-man's-switch** Prometheus emits a Watchdog heartbeat to
  Healthchecks.io. If the lab goes dark, Healthchecks.io raises an
  independent alert.
- **One pane of glass** Grafana. Metrics, logs, and heartbeat all
  share the same UI. Wazuh has its own dashboard for SIEM workflows.

## Deep dive

### Metrics

Prometheus is deployed via `kube-prometheus-stack`. Scrape targets:

- **In-cluster** everything kube-prometheus-stack knows about out of
  the box (kubelet, kube-state-metrics, node-exporter daemonset on
  Kubernetes nodes), plus every `ServiceMonitor` that comes with a
  workload.
- **VMs** node_exporter on every non-Kubernetes VM, discovered via a
  Prometheus **file_sd** ConfigMap that the Ansible monitoring
  playbook regenerates from dynamic inventory. The inventory is sourced
  from the Proxmox API, so a new VM shows up in Prometheus without a
  manual scrape-config edit.
- **Proxmox** pve-exporter on each hypervisor, returning per-VM and
  per-node metrics. Both exporters return cluster-wide data, which
  requires care on dedupe.
- **TrueNAS** scraped at the standard exporter endpoint. The
  `instance` label is overridden to `truenas` so dashboards do not
  drift when the scrape address changes.

Textfile collectors on VMs let Ansible emit custom metrics (reboot
timers, unattended-upgrades outcomes) without needing a separate
exporter per concern.

### Logs

**Loki (SingleBinary)** in Kubernetes, backed by an NFS-mounted PV.
30-day retention. No object store backing at this scale; if log
volume grows I can swap to S3/MinIO without touching client config.

**Vector** ships from two places, not three:

- **Vector DaemonSet** in Kubernetes reads container logs, excludes
  `kube-system` and `monitoring` namespaces (they are noisy without
  being useful), and pushes to Loki over its in-cluster Service.
- **Vector agent** on every non-Kubernetes VM, running as a systemd
  service, shipping journald and `/var/log/**/*.log` to Loki via
  its external LoadBalancer IP.

VRL transforms at the Vector layer do two kinds of work:

- **Noise drop.** NVIDIA driver chatter and similar known-useless
  lines are filtered at ingest so Loki's label cardinality stays
  sane. Cheaper than filtering at query time.
- **Label shaping.** The `app` label is extracted from log paths so
  dashboards can split by app without expensive regex per query.

For the full rationale behind this shape (and why it is not Promtail)
see [`elastic-to-loki-migration.md`](../lessons-learned/elastic-to-loki-migration.md).

### SIEM: Wazuh alongside Loki

Loki is a log aggregator. Wazuh is a SIEM. They have overlap on "I
can put text in and find text later" but the jobs they do are
different. Wazuh handles:

- **File integrity monitoring** on every Linux host.
- **Rootkit detection, vulnerability scanning, system call
  auditing** via native agents.
- **Compliance-shaped rulesets** (CIS, PCI, etc.) out of the box.

Wazuh runs as an all-in-one install (Manager + Indexer + Dashboard)
on its own VM. Agents are grouped by role: `linux`, `kubernetes`,
`proxmox`. The dashboard is its own UI; Grafana does not try to be a
SIEM.

### Alerting

**Alertmanager is the only alert router.** Grafana alerting is
disabled cluster-wide. One router, one inhibition tree, one silence
UI. This is not a dogma; it is the lesson from years of running two
overlapping alert systems and paying tax on reconciling them.

**Discord is the notification channel.** Alertmanager webhooks to an
in-cluster `discord-alert-proxy` (baked immutable image, two
replicas, CI-scanned) which formats alerts with themed styling:

- LOTR palette for critical/red.
- Star Wars palette for warning/orange.
- Fantasy palette for resolved/green.

The theming is not cuteness. The shape of a "critical Gandalf" Discord
embed is recognisable at 2am faster than a monochrome template, and
in practice that matters for time-to-ack.

**HA Alertmanager** runs two replicas with soft pod anti-affinity and
a PodDisruptionBudget. `PrometheusNotificationsDropped` itself is an
alert, so a stuck queue pages. The story behind this posture is
[`alertmanager-spof.md`](../lessons-learned/alertmanager-spof.md).

### Dead-man's-switch

Every monitoring system is a liar unless you prove it is alive.
Prometheus emits a Watchdog alert on a timer; Alertmanager routes
that alert to **Healthchecks.io**. Healthchecks.io expects a heartbeat
at a known cadence. If the heartbeat stops, Healthchecks.io raises its
own independent alert, reachable through a channel that does not
depend on anything in the lab.

This is the "I would notice if alerts stopped" trap, solved in the
right place. The monitor cannot be inside the thing it monitors.

### Dashboards

Grafana is the single UI for metrics and logs. Dashboards live in
[`observability-quasarlab`](https://github.com/mithr4ndir/observability-quasarlab)
and are provisioned via ConfigMaps. Highlights:

- **Cluster overview** nodes, pods, namespaces, resource pressure.
- **VM fleet** node_exporter health per VM, grouped by hypervisor.
- **Proxmox** per-host and per-guest health, HA status, storage usage.
- **Jellyfin + GPU** active streams, transcode metrics, NVIDIA GPU
  utilization and recovery counters. Real-time behaviour of the
  lab's most-user-facing workload.
- **Log explorers** VM and Kubernetes log explorers with host,
  app, namespace, and pod filters.
- **1Password quota** live `USED`/`LIMIT` from
  `op service-account ratelimit`, alerting on utilization so the
  Phase 2 secrets rollout does not sneak up on the rate limit.

### Workflow: what happens when something breaks

A typical investigation at 2am:

1. Discord pings with a themed alert. Severity and component are
   visible before I open the app.
2. The alert annotation includes a runbook URL and a Grafana link
   pre-filtered to the right dashboard and time window.
3. If the alert is about logs (rare but possible), Grafana's log
   explorer opens with the affected component and time range.
4. If the alert is about security (FIM trigger, rootkit signature),
   Wazuh's dashboard is the destination.
5. Silences go in Alertmanager, not Grafana. One place. Always.

### Related code

- [`observability-quasarlab`](https://github.com/mithr4ndir/observability-quasarlab)
  Prometheus rules, Loki config, Grafana dashboards, Vector
  configuration.
- [`k8s-argocd`](https://github.com/mithr4ndir/k8s-argocd) the
  Helm releases that put the stack on the cluster.
- [`ansible-quasarlab`](https://github.com/mithr4ndir/ansible-quasarlab)
  VM-side Vector and Wazuh agents, node_exporter with textfile
  collectors, and the Prometheus file_sd sync script.
