# Security

Security posture, public-exposure strategy, and hygiene for this repo and
the lab it documents.

## What is here

- [public-showcase.md](public-showcase.md) design for the read-only public
  Grafana instance behind Cloudflare Tunnel.
- Commit hygiene: gitleaks + pre-commit enforced locally and in CI.
- Reporting: see top-level [SECURITY.md](../../SECURITY.md).

## Principles

- Default-deny at every layer: network, RBAC, secrets.
- Secrets never in git. 1Password + External Secrets Operator only.
- Internet exposure is explicit, narrowly scoped, and documented here.
- Defense in depth: WAF, tunnel auth, app-level anonymous viewer, read-only
  datasource. Any single layer failure still has the next one behind it.
