# Alerts

Alertmanager routing, Prometheus rules, and Discord formatting.

## What is here

- Routing tree overview (priorities, inhibitions, silences). TODO.
- Discord message conventions (themed, structured, scannable).
- Dead man's switch design.

## Conventions

- Alertmanager is the one router. Grafana alerting is not used.
- Every alert has a runbook URL. No runbook, no alert.
- Alerts carry enough labels for automatic routing but no sensitive
  content in the annotation body.
- Themed formatting (LOTR, Star Wars) is a triage aid, not decoration.

TODO paste the sanitised routing YAML and an example Discord embed.
