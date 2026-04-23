# Kubernetes automation

Notes on how the cluster is provisioned and kept in GitOps shape.

## What is here

- ArgoCD app-of-apps and ApplicationSet patterns.
- Helm + Kustomize rendering conventions.
- Secret flow via 1Password + External Secrets Operator.

## Conventions

- `values.yaml` is the source of truth. Rendered YAML is generated in
  CI and is ignored in git.
- Every app belongs to a sync wave.
- Alerts fire through Alertmanager to Discord. Never through Grafana.

TODO link a representative ApplicationSet and the render pipeline.
