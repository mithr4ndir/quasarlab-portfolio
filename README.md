# quasarlab-portfolio

A public showcase of the quasarlab homelab: infrastructure-as-code, GitOps,
observability, and the lessons learned while running it.

This repo is documentation only. The code that runs the lab lives in sibling
repositories (`ansible-quasarlab`, `terraform-quasarlab`, `k8s-argocd`,
`observability-quasarlab`, and others). Those repos are private; this one is
public and curated for portfolio and community use.

## Stack

| Layer         | Tools                                                                         |
|---------------|-------------------------------------------------------------------------------|
| Compute       | Proxmox (HA pair), Kubernetes (3 control-plane nodes, HAProxy/Keepalived VIP) |
| Storage       | TrueNAS, NFS, PostgreSQL                                                      |
| Observability | Prometheus, Alertmanager, Grafana, Loki + Vector, Wazuh (SIEM)                |
| GitOps        | ArgoCD, Helm rendered via Kustomize, values.yaml as source of truth           |
| Automation    | Ansible, Terraform                                                            |
| Secrets       | 1Password + External Secrets Operator                                         |
| Edge          | Cloudflare Tunnel, Nginx Proxy Manager                                        |

> Logging was previously on the Elastic Stack. The lab moved to Loki + Wazuh;
> the reasoning is documented in
> [lessons-learned/elastic-to-loki-migration.md](lessons-learned/elastic-to-loki-migration.md).

## Highlights

A few stories worth reading first. Each is a filled-in retrospective,
not a placeholder:

- [Elastic to Loki + Vector + Wazuh migration](lessons-learned/elastic-to-loki-migration.md)
  Full log-platform swap. Parallel run, clean cutover, complete
  decommissioning. Freed tens of GBs of RAM and collapsed two panes
  of glass into one.
- [Proxmox HA fence, nine hours of silence](lessons-learned/pve-ha-outage.md)
  A real production-shape incident. HA fenced a node, VMs didn't come
  back, and the alert path was quietly broken. Four-day remediation
  across 21 merged PRs: boot-race fix, alert regex anchoring, 1Password
  rate-limit hardening, etcd defrag automation, Kubernetes upgrade, and
  an external dead-man's-switch.
- [Secrets as code on 1Password, and what the rate limit taught us](lessons-learned/secrets-iac-1password-eso.md)
  Defense-in-depth for a rate-limited secret backend: caching, kill
  switch, scoped service accounts, Prometheus quota observability,
  ESO circuit breaker, explicit rollout ordering. Includes the "the
  loud failure is not always the cause" bit.
- [Alertmanager single point of failure](lessons-learned/alertmanager-spof.md)
  A six-hour outage with zero alerts delivered. HA Alertmanager +
  PDB + external heartbeat. "I would notice" is not a monitoring
  strategy.
- [Grafana VM to Kubernetes migration](lessons-learned/grafana-k8s-migration.md)
  Moved state (SQLite to PostgreSQL via pgloader), moved workload
  (kube-prometheus-stack Helm), moved clients (MetalLB + NPM), and
  decommissioned the old VM by the full checklist.

## Repository map

- [architecture/](architecture/) high-level diagrams and design notes
- [docs/](docs/) per-domain deep dives (kubernetes, observability, storage,
  media stack, astronomy, security)
- [lessons-learned/](lessons-learned/) post-incident writeups and design
  retrospectives
- [dashboards/](dashboards/) curated Grafana dashboards and screenshots
- [alerts/](alerts/) Alertmanager routing, Prometheus rules, Discord patterns
- [automation/](automation/) notes on the Ansible, Terraform, and Kubernetes
  toolchains
- [screenshots/](screenshots/) static evidence for the portfolio

## Live showcase

TODO. A read-only public Grafana instance will be published behind a
Cloudflare Tunnel. The design lives in
[docs/security/public-showcase.md](docs/security/public-showcase.md).

## Open-source contributions

Work on astronomy and data tooling that came out of this lab:

- [lsdb#1325](https://github.com/astronomy-commons/lsdb/pull/1325) Kubernetes
  deployment documentation
- [lightkurve#1553](https://github.com/lightkurve/lightkurve/pull/1553) CI
  improvements
- [yt#4391](https://github.com/yt-project/yt/pull/4391) documentation CI
  workflow

## Contact

- GitHub: [@mithr4ndir](https://github.com/mithr4ndir)
- LinkedIn: TODO

## License

MIT. See [LICENSE](LICENSE).
