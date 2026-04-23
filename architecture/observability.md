# Observability architecture

TODO. Describe:

- Metrics: Prometheus (scrape targets, service discovery, retention).
- Logs: Loki with a Vector agent + DaemonSet + Aggregator pipeline for
  application, platform, and external syslog. Wazuh for SIEM/FIM.
- Traces: TODO (document if/when added).
- Alerts: Alertmanager routing to Discord. Grafana alerting is not used.
- Dashboards: Grafana, versioned in `observability-quasarlab`.
- Dead-man's-switch: Watchdog alert + external probe.
