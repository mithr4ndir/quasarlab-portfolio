# Kubernetes

Operational notes for the quasarlab Kubernetes cluster.

## What is here

- Cluster topology and control-plane HA notes.
- ArgoCD patterns (app-of-apps, ApplicationSets, sync waves).
- Helm + Kustomize rendering conventions.
- Upgrade runbook references.

## Conventions

- Helm charts are rendered via Kustomize overlays. `values.yaml` is the
  source of truth; rendered YAML is ignored.
- Every workload has an ArgoCD Application and belongs to a sync wave.
- Secrets are never committed. 1Password items are referenced through
  ExternalSecret resources managed by External Secrets Operator.
- Alerts fire through Alertmanager to Discord. Grafana alerting is not used.

TODO fill in cluster-specific details and link to the private repos.
