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
| Observability | Prometheus, Alertmanager, Grafana, Loki + Promtail, Wazuh (SIEM)              |
| GitOps        | ArgoCD, Helm rendered via Kustomize, values.yaml as source of truth           |
| Automation    | Ansible, Terraform                                                            |
| Secrets       | 1Password + External Secrets Operator                                         |
| Edge          | Cloudflare Tunnel, Nginx Proxy Manager                                        |

> Logging was previously on the Elastic Stack. The lab moved to Loki + Wazuh;
> the reasoning is documented in
> [lessons-learned/elastic-to-loki-migration.md](lessons-learned/elastic-to-loki-migration.md).

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
