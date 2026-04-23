# Kubernetes architecture

## TL;DR

- Three-node cluster, every node runs the control plane and workloads.
  Stacked etcd. HA API endpoint via **kube-vip**.
- All workloads are declared in [`k8s-argocd`](https://github.com/mithr4ndir/k8s-argocd)
  and reconciled by **ArgoCD** using an **app-of-apps** pattern. Sync
  waves order dependencies.
- Helm charts are **rendered through Kustomize**. `values.yaml` is the
  source of truth; rendered YAML is ignored in git.
- Secrets flow **1Password -> External Secrets Operator -> Kubernetes
  Secret**. No plaintext secrets in any repo, public or private.
- External LoadBalancer IPs come from a **MetalLB** pool. Cluster-
  external consumers (NPM, Cloudflare Tunnel) talk to stable IPs, not
  cluster-internal Services.

## Deep dive

### Topology

Three-node cluster where every node is a control-plane node and also
runs workloads. In a bigger environment I would separate control plane
from workers; at homelab scale the extra VMs add more operational
cost than the separation is worth.

- **etcd** is stacked on the control-plane nodes. Weekly defrag timer
  runs via the `k8s_maintenance` Ansible role (Sunday 04:00), added
  after the [Proxmox HA outage](../lessons-learned/pve-ha-outage.md)
  where an etcd DB had grown large enough to matter.
- **Kubernetes API** is fronted by **kube-vip** providing a single
  highly-available VIP. `kubectl`, ArgoCD, and every in-cluster
  controller talk to that VIP. When one control-plane node goes
  down, the VIP fails over to a surviving node and nothing at the
  API level notices.
- **CNI**: Kubernetes default pod CIDR (`10.244.0.0/16`). Flannel-class
  network, simple L3 for pod-to-pod.

### GitOps: ArgoCD app-of-apps

The entire cluster is declarative from `k8s-argocd`. Nothing ships
via `kubectl apply`. The pattern:

- A root `Application` points at an `environments/prod/` path.
- That path contains per-domain `Application` resources
  (`infrastructure/monitoring`, `media`, `science`, `security`, etc.).
- Each of those points at its own directory of manifests.

Sync waves order dependencies: infrastructure (MetalLB, cert-manager,
External Secrets Operator, NetworkPolicies) syncs first, then
secrets backends, then tenant namespaces, then workloads.

`ApplicationSet` generates per-workload Applications where the shape
is repeated (for example, many similar media-stack apps).

This means:

- **Audit is git log.** Every change to the cluster is a PR with a
  review trail.
- **Rollback is git revert.** ArgoCD reconciles back.
- **Drift detection is free.** ArgoCD flags anything modified
  out-of-band.

### Helm rendered through Kustomize

This is the one unusual choice, and it is deliberate.

Most teams pick one: either Helm (for upstream charts) or Kustomize
(for their own manifests). I do both at once: every upstream chart is
rendered into YAML via `helm template`, and the rendered YAML is
fed into a Kustomize overlay that applies lab-specific patches.

What this buys:

- **`values.yaml` is the source of truth** for every workload. Anyone
  reading the repo sees exactly what changed from the upstream chart.
- **Upstream bumps are reviewable.** When a chart moves from v1 to
  v2, the diff in rendered YAML tells me exactly what broke or
  moved.
- **Policy hooks fit cleanly.** Kustomize transformers, labelers, and
  commonAnnotations apply uniformly across Helm-originated and
  hand-written manifests.

Cost: a rendering step in CI (and locally via a make target). The
rendered output is generated, not committed (covered by `.gitignore`
entries for `*.rendered.yaml` and `rendered/`).

### Secrets: 1Password + External Secrets Operator

No Kubernetes Secret is authored by hand or committed anywhere.

- **1Password** is the source. The lab has scoped service accounts
  per consumer.
- **External Secrets Operator** watches `ExternalSecret` resources in
  the cluster and materialises them into native Kubernetes Secrets.
- Workloads consume normal `Secret` objects and never know 1Password
  was involved.
- `ClusterSecretStore` resources reference the operator's service
  account; every workload namespace has its own `ExternalSecret`
  declarations in git.

The two big operational lessons from this pipeline live in
[`secrets-iac-1password-eso.md`](../lessons-learned/secrets-iac-1password-eso.md):
rate-limit scope is per account not per service account, and the
loud failure (ESO spam) is rarely the cause of a rate-limit drain.

### Workloads and namespaces

Namespaces correspond to domains, not teams. Rough map:

- `monitoring` kube-prometheus-stack, Loki, Vector DaemonSet,
  Alertmanager, discord-alert-proxy, Grafana.
- `external-secrets` ESO and its CRDs.
- `media` Jellyfin, the *arr apps, NZBGet, Jellyseerr.
- `science` Dask Kubernetes operator and batch workloads for
  astronomy contributions.
- `security` Falco, Trivy Operator.
- `nfs` NFS subdir external provisioner for dynamic PVs on TrueNAS.
- `metallb-system` LoadBalancer IP allocation.
- `reloader` ConfigMap/Secret reload controller.
- `dashboard` Homepage (gethomepage) as the lab landing page.
- `dev` scratch namespace for experiments.
- `trading` a contained tenant for an in-house signal engine.

Each namespace is its own ArgoCD Application tree; they can be
synced, paused, and rolled back independently.

### Reliability posture

A few non-obvious choices that come out of real incidents:

- **Alertmanager runs two replicas with a PDB** (minAvailable: 1) and
  soft pod anti-affinity so node drains cannot take the alert path
  to zero. The original lesson is
  [`alertmanager-spof.md`](../lessons-learned/alertmanager-spof.md).
- **Discord alert proxy is HA, baked immutable image, Dependabot +
  Trivy in CI.** The proxy is a critical link; it gets the same
  reliability budget as the thing it routes for.
- **`kubectl rollout restart` is disallowed by operating rule for
  *arr apps on NFS-backed SQLite.** Use scale-0-then-1 instead. See
  [`arr-rollout-restart.md`](../lessons-learned/arr-rollout-restart.md).
- **Watchdog is routed to Healthchecks.io**, not to `null`, so a
  dead Prometheus-or-cluster shows up as an external alert even
  when the lab's own alert path is compromised.

### Upgrade discipline

Rolling upgrades on the control plane, one node at a time, after
the etcd weekly defrag has run. The last wave (v1.33.11 + kernel
6.8.0-110) landed during the `pve-ha-outage` remediation window
and is documented there.

### Related code

- [`k8s-argocd`](https://github.com/mithr4ndir/k8s-argocd) every
  manifest, every ApplicationSet, every `values.yaml`.
- [`observability-quasarlab`](https://github.com/mithr4ndir/observability-quasarlab)
  dashboards and alert rules that drop into the cluster's Grafana and
  Prometheus.
- [`ansible-quasarlab`](https://github.com/mithr4ndir/ansible-quasarlab)
  handles the control-plane hosts' OS layer, kubelet/containerd
  upgrades, and etcd defrag automation.
