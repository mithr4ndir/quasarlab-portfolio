# Observability

The observability stack combines metrics, logs, and SIEM in one Grafana
pane of glass, with Alertmanager as the single alert router.

## What is here

- How Prometheus, Loki, Vector, Grafana, and Wazuh fit together.
- Alert routing conventions and Discord formatting.
- Dashboard inventory (linked from `dashboards/`).

## Logging infrastructure

- Loki (SingleBinary) is the primary log store.
- Vector is the collector. Two shapes, one tool:
  - Vector DaemonSet in Kubernetes for container logs, with kube-system and
    monitoring namespaces excluded to keep Loki's label cardinality sane.
  - Vector agent on every non-Kubernetes VM, shipping `journald` and
    `/var/log/**/*.log` to Loki.
- Wazuh for security-focused log collection, file integrity monitoring, and
  SIEM use cases. Runs alongside Loki, not replacing it.
- Grafana is the single UI for logs and metrics. Loki is a Grafana
  datasource; Wazuh has its own dashboard for SIEM workflows.
- Dashboards ship LogQL explorers with host/log-type/app filters for VMs
  and namespace/app/pod filters for Kubernetes. The `app` label is
  extracted from log paths so scan-time queries stay fast.
- Retention: 30 days at the Loki layer; sensitive events land in Wazuh
  with its own retention tier.
- Log routing conventions: application and platform logs go to Loki;
  security-relevant events (auth, FIM, integrity) are handled by Wazuh
  agents; GPU driver chatter and other known noise is dropped at the
  Vector layer rather than filtered at query time.

## Alerting

- Alertmanager is the single source of truth for routing.
- Discord is the primary notification channel. Alerts are themed
  (LOTR/Star Wars flavour) for human readability, not for cuteness: a
  recognisable shape is easier to triage at 2am.
- Grafana alerting is intentionally disabled. One router, one inhibition
  tree, one silence UI.
- Watchdog alert + external probe form the dead man's switch.

## Related repos

- `observability-quasarlab` Prometheus rules, Loki config, Grafana dashboards.

TODO expand with screenshots and dashboard walk-throughs.
