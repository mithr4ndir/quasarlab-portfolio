# QuasarLab Portfolio Onboarding Guide

Welcome. This repo is the documentation half of a homelab. There is no
application to run, no test suite to pass, and no service to deploy. What
you contribute here is prose, diagrams, retrospectives, and the occasional
sanitised config snippet. This guide tells you what the repo is, how it is
laid out, the vocabulary you need to read it fluently, and what it looks
like to make a change.

---

## What Is This?

`quasarlab-portfolio` is the public-facing documentation for **quasarlab**,
a personal homelab built around Proxmox, Kubernetes, GitOps, and a
metrics-plus-logs-plus-SIEM observability stack. The owner is an
infrastructure and platform engineer; the repo is shaped to read like a
working portfolio rather than a tutorial.

The code that runs the lab lives in separate public repositories
(`ansible-quasarlab`, `terraform-quasarlab`, `k8s-argocd`,
`observability-quasarlab`). This repo is the curated narrative that ties
those code repos together: architecture deep-dives, post-incident
retrospectives, design decisions, and screenshots from the live lab. The
two private siblings (`truenas-config-backup`,
`quasarlab-disaster-recovery`) stay private because they hold credentials
and DR runbooks.

Read [`README.md`](README.md) first for the project's own framing. The
flagship retrospectives in [`lessons-learned/`](lessons-learned/) are
where most readers should start after the README.

---

## Reader Experience

The audience here is people evaluating the work: hiring managers,
recruiters, peers, and the curious. There are three shapes of reader path,
and the repo is laid out to serve all three:

- **Skimmer.** Lands on `README.md`, scans the stack table, the "how it
  fits together" Mermaid diagram, the field-reports list, and leaves
  with a feel for what the lab does. Total time: under five minutes.
- **Architecture reader.** Wants to understand a specific layer
  (network, Kubernetes, observability, data). Heads to
  [`architecture/`](architecture/) and reads one or two deep-dives.
  Each file has a TL;DR up top so they can decide whether to keep
  reading.
- **Postmortem reader.** Wants to see how the lab handles real
  incidents. Heads to [`lessons-learned/`](lessons-learned/) and reads
  one of the flagship writeups. Each follows the same template (Date,
  Impact, Duration, Summary, Background, What happened, Investigation,
  Root cause, Fix, Takeaways, Follow-ups).

Visual content lives in [`screenshots/`](screenshots/) (live Grafana and
ARA captures) and [`dashboards/`](dashboards/) (curated dashboard
exports). Both are linked from `README.md` and from individual
retrospectives where relevant.

A read-only public Grafana instance is planned but not yet live; the
design is documented in
[`docs/security/public-showcase.md`](docs/security/public-showcase.md).

---

## How Is It Organized?

This is a documentation repo. There is no build, no entry point, and no
runtime. The "architecture" is the directory layout and the editorial
conventions that govern what goes where.

### Repo layout

```
quasarlab-portfolio/
  README.md            # Front door, stack table, field reports
  SECURITY.md          # Vulnerability reporting, secret hygiene
  architecture/        # Per-axis deep dives (network, k8s, etc.)
  docs/                # Per-domain notes (kubernetes, security, etc.)
  lessons-learned/     # Post-incident writeups and retrospectives
  dashboards/          # Curated Grafana dashboards (JSON, PNG)
  alerts/              # Alertmanager routing, Discord conventions
  automation/          # Notes on Ansible, Terraform, K8s tooling
  screenshots/         # Live-lab captures with caption walkthrough
  assets/              # README hero image and other static assets
  .github/workflows/   # Gitleaks scanning in CI
```

### Where each kind of content lives

| Directory | Responsibility |
|-----------|----------------|
| `architecture/` | High-level design notes, one file per axis (network, kubernetes, observability, data). Each starts with a TL;DR. |
| `docs/` | Per-domain operational notes (kubernetes, observability, storage, security, media-stack, astronomy). |
| `lessons-learned/` | Incident retrospectives and design retrospectives, all using a shared template. |
| `automation/` | Conventions for the toolchains (Ansible, Terraform, Kubernetes). The actual code lives in sibling repos. |
| `alerts/`, `dashboards/`, `screenshots/` | Sanitised exports and captures from the live observability stack. |
| `assets/` | Static images for the README and other Markdown files. |

### How the pieces connect

`README.md` is the front door and links into every other directory.
Architecture deep-dives in `architecture/` link forward to relevant
retrospectives in `lessons-learned/` (for example,
`architecture/kubernetes.md` links to the Proxmox HA outage and the
Alertmanager SPOF writeups). Retrospectives link back to architecture
docs for context. Sibling code repos are linked from both layers when a
specific module is responsible for the behaviour being described.

There is no canonical reading order beyond "start at the README." The
graph of links is dense on purpose: any starting page should be one or
two hops away from the answer.

### External dependencies

Even though there is no runtime, the repo has real external
dependencies in two senses: the sibling code repos it documents, and
the tooling that enforces commit hygiene.

| Dependency | What it is | Where it shows up |
|-----------|-----------|-------------------|
| [`ansible-quasarlab`](https://github.com/mithr4ndir/ansible-quasarlab) | Host config, inventory, playbooks for the lab | Linked from `architecture/`, `automation/ansible/`, lessons |
| [`terraform-quasarlab`](https://github.com/mithr4ndir/terraform-quasarlab) | VM, Proxmox, Cloudflare resource provisioning | Linked from architecture and automation docs |
| [`k8s-argocd`](https://github.com/mithr4ndir/k8s-argocd) | ArgoCD app-of-apps, every workload manifest | Linked from `architecture/kubernetes.md` and most lessons |
| [`observability-quasarlab`](https://github.com/mithr4ndir/observability-quasarlab) | Prometheus rules, Loki config, Grafana dashboards | Linked from `architecture/observability.md`, `dashboards/` |
| `truenas-config-backup` (private) | Scheduled TrueNAS config snapshots | Mentioned in `architecture/data.md` |
| `quasarlab-disaster-recovery` (private) | DR runbooks | Mentioned in `docs/security/` |
| `gitleaks` | Secret scanning, runs in pre-commit and CI | `.gitleaks.toml`, `.github/workflows/gitleaks.yml`, `.pre-commit-config.yaml` |
| `pre-commit` | Trailing whitespace, EOF, YAML/JSON syntax, large files, private keys | `.pre-commit-config.yaml` |
| `yamllint` | YAML style enforcement (relaxed ruleset) | `.pre-commit-config.yaml` |

---

## Key Concepts and Abstractions

The repo's vocabulary comes from the lab itself. You do not need to know
every term before reading, but the ones below show up often enough that a
glossary is worth keeping near.

| Concept | What it means in this codebase |
|---------|--------------------------------|
| **quasarlab** | The homelab being documented. All public siblings carry the `-quasarlab` suffix. |
| **Field report** | A flagship retrospective in `lessons-learned/`. Filled in, ready to read, as opposed to a stub. |
| **Lessons-learned template** | The shared structure for every retrospective: Date, Impact, Duration, Category, Summary, Background, What happened, Investigation, Root cause, Fix, Takeaways, Follow-ups. |
| **TL;DR convention** | Architecture deep-dives in `architecture/` open with a bulleted TL;DR before the prose. |
| **No-real-values rule** | Hostnames, IPs, MAC addresses, account IDs, and serials never appear in this repo. Describe the pattern, not the value. Enforced by review and by `.gitleaks.toml`. |
| **Two edges** | The lab has Nginx Proxy Manager (LAN) and Cloudflare Tunnel (public). Public surface is one read-only hostname. |
| **app-of-apps** | ArgoCD pattern where one root `Application` points at a directory of per-domain `Application` resources. The cluster is declarative from `k8s-argocd`. |
| **Helm rendered through Kustomize** | Upstream Helm charts rendered via `helm template`, then patched by Kustomize overlays. `values.yaml` is the source of truth; rendered YAML is `.gitignore`d. |
| **Sync wave** | ArgoCD ordering primitive used to sync infrastructure before secrets backends before workloads. |
| **ESO** | External Secrets Operator. Reads from 1Password, materialises Kubernetes Secrets. Workloads never see 1Password directly. |
| **Themed alerts** | Alertmanager routes to a custom in-cluster Discord proxy that styles alerts (LOTR for critical, Star Wars for warning, fantasy for resolved). Recognisability is a triage aid. |
| **Dead man's switch** | Prometheus emits a Watchdog heartbeat to Healthchecks.io. If the lab goes dark, Healthchecks.io raises an independent alert. |
| **scale-0-then-1** | The operating rule for *arr apps on NFS-backed SQLite. Never `kubectl rollout restart`. |
| **Vector, two shapes** | One Vector DaemonSet in Kubernetes plus a Vector systemd agent on every non-K8s VM. Both ship to Loki. |
| **file_sd** | Prometheus static-targets ConfigMap regenerated by Ansible from dynamic Proxmox-API inventory. |

A few editorial abstractions are worth naming explicitly because they
shape what gets accepted:

- **Architecture deep-dives are abstract by design.** Real IPs, VLANs,
  and topology stay in the private DR repo. Public docs describe the
  shape.
- **Retrospectives are honest.** "What broke and why" includes the
  embarrassing parts. The Proxmox HA outage and Alertmanager SPOF
  writeups are the worked examples.
- **Stubs are labelled.** Files marked TODO or `status-stub` are
  acknowledged gaps, not finished work pretending to be finished.

---

## Primary Flows

The repo has two interesting flows: a contributor flow (someone making
a change) and a reader flow (someone consuming the content).

### Contributor flow

```
Edit Markdown / images
  |
  v
pre-commit hooks run locally
  trailing-whitespace, end-of-file-fixer
  check-yaml, check-json
  detect-private-key, check-added-large-files (4096 KB cap)
  gitleaks (per .gitleaks.toml)
  yamllint (relaxed)
  |
  v
git commit / push
  |
  v
.github/workflows/gitleaks.yml runs in CI
  on push to main, every PR, and weekly cron
  |
  v
Review and merge
```

The pre-commit configuration lives in
[`.pre-commit-config.yaml`](.pre-commit-config.yaml). The CI workflow is
[`.github/workflows/gitleaks.yml`](.github/workflows/gitleaks.yml). A
gitleaks finding fails the build; clear it before merging.

### Reader flow

```
Land on README.md
  |
  +--> Stack table + Mermaid topology diagram
  |
  +--> Field reports list (lessons-learned/*.md)
  |       |
  |       v
  |    Specific retrospective using the shared template
  |       (links back into architecture/ for context)
  |
  +--> Repository map
          |
          +--> architecture/  per-axis deep-dives
          +--> docs/          per-domain notes
          +--> dashboards/    curated exports
          +--> screenshots/   live-lab captures
          +--> alerts/        routing and Discord
          +--> automation/    toolchain notes
```

Most readers visit two or three pages and leave. The architecture and
retrospective layers are written so each page stands alone with TL;DR
and inline links.

---

## Developer Guide

### Setup

You need a working local clone, Git, and `pre-commit` installed once.
Everything else is Markdown and images.

```
git clone https://github.com/mithr4ndir/quasarlab-portfolio.git
cd quasarlab-portfolio
pipx install pre-commit
pre-commit install
```

To run all hooks against the whole repo (useful before a big PR):

```
pre-commit run --all-files
```

### Common change patterns

- **Add a new retrospective.** Create
  `lessons-learned/<slug>.md`. Follow the template documented in
  [`lessons-learned/README.md`](lessons-learned/README.md): Date,
  Impact, Duration, Category, Summary, Background, What happened,
  Investigation, Root cause, Fix, Takeaways, Follow-ups. Add an entry
  to `lessons-learned/README.md` and to the field-reports list in
  the top-level `README.md`. Mark its status with a shields.io badge
  (`status-resolved`, `status-stub`, `design-in-flight`,
  `rule-in-effect`).
- **Add or refresh an architecture deep-dive.** Pick the right axis
  (`network`, `kubernetes`, `observability`, `data`) and edit the
  matching file under `architecture/`. Open with a TL;DR. Cross-link
  back to relevant retrospectives. If the topic is new, create a new
  file and add it to `architecture/README.md`.
- **Add a domain doc.** Create or edit a folder under `docs/`
  (`docs/<domain>/README.md` is the convention). Domain docs are the
  operational layer; architecture docs are the design layer. Linkable
  from the architecture file when a deep-dive references operations.
- **Add a screenshot.** Drop a PNG into `screenshots/`. Update
  `screenshots/README.md` with the caption (live Grafana and ARA
  captures only; PNG only; under 4096 KB).
- **Add a dashboard or alert export.** Drop sanitised JSON in
  `dashboards/` or `alerts/`. Real hostnames, IPs, alert content, and
  recording-rule labels are removed before commit.
- **Reference sibling repos.** Link to the GitHub URL inline rather
  than copying code over. The sibling repo is the source of truth for
  any code; this repo is the source of truth for narrative.

### Editorial rules

- **No real values.** Hostnames, IPs, MAC addresses, serials, account
  IDs, tokens. Use `example.internal`, `10.0.0.0/8` only as abstract
  shapes, or describe the pattern instead of the value.
- **Every alert documented here has a runbook URL.** No runbook, no
  alert. (See [`alerts/README.md`](alerts/README.md).)
- **Sibling-repo links use the public GitHub URL.** Private siblings
  are mentioned by name and described, never linked.
- **Status badges are honest.** A stub is labelled
  `status-stub`. A flagship retrospective is `status-resolved` or
  `design-in-flight`. Do not call something resolved if it is not.
- **Hero image and other AI-generated assets** stay under `assets/`
  and inherit the repo's MIT license.

### Practical tips

- The flagship retrospectives in `lessons-learned/` are the densest
  reading; they are also the strongest portfolio surface. When in
  doubt about how much detail to include in a new writeup, read
  [`lessons-learned/pve-ha-outage.md`](lessons-learned/pve-ha-outage.md)
  for the worked example.
- The four `architecture/*.md` deep-dives all open with a TL;DR. If
  you are editing one, keep that pattern; readers rely on it.
- The Mermaid diagram in `README.md` is the canonical "how it fits
  together" picture. If a system-shape change happens (a new edge, a
  new datastore), update that diagram in the same PR as the
  architecture file.
- `gitleaks` will catch the obvious mistakes (tokens, private keys),
  but it does not catch hostnames or IPs. Re-read your diff before
  pushing if you have been working in private docs and might have
  copy-pasted a real value.
