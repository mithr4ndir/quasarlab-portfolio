# Observability architecture

TODO. Describe:

- Metrics: Prometheus (scrape targets, service discovery, retention).
- Logs: Loki + Promtail for application and platform, Wazuh for SIEM/FIM.
- Traces: TODO (document if/when added).
- Alerts: Alertmanager routing to Discord. Grafana alerting is not used.
- Dashboards: Grafana, versioned in `observability-quasarlab`.
- Dead-man's-switch: Watchdog alert + external probe.
