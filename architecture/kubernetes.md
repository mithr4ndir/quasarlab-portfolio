# Kubernetes architecture

TODO. Describe:

- Control plane: 3 nodes, stacked etcd, HAProxy + Keepalived VIP for API.
- Worker pool composition and node labels/taints.
- Ingress controller, cert-manager, ExternalDNS (if used).
- GitOps: ArgoCD app-of-apps, ApplicationSets, sync policy.
- Helm + Kustomize rendering: values.yaml is the source of truth, rendered
  output is ignored via .gitignore.
- Secrets flow: 1Password + External Secrets Operator.
