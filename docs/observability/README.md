# Observability

The observability stack combines metrics, logs, and SIEM in one Grafana
pane of glass, with Alertmanager as the single alert router.

## What is here

- How Prometheus, Loki, Promtail, Grafana, and Wazuh fit together.
- Alert routing conventions and Discord formatting.
- Dashboard inventory (linked from `dashboards/`).

## Logging infrastructure

- Loki + Promtail as the primary log pipeline for application and platform logs.
- Wazuh for security-focused log collection, file integrity monitoring, and
  SIEM use cases. Runs alongside Loki, not replacing it.
- Grafana is the single UI for logs and metrics. Loki is a Grafana
  datasource; Wazuh has its own dashboard for SIEM workflows.
- Retention tiering: TODO (document hot/warm/cold split, S3/MinIO backing if used).
- Log routing conventions: TODO (what goes to Loki, what stays in Wazuh,
  what gets dropped at the agent).

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
