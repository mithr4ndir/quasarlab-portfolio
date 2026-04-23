# Screenshots

A short visual tour of the lab. Captures come from the live Grafana
and ARA instances.

## Tour

### QuasarLab overview

![QuasarLab overview dashboard showing VM health, status table, and resource charts](quasarlab-overview.png)

Top-level lab health: VMs online, pending reboots, failed upgrades,
security update counters, and the "Ansible Timer OK" tile. The status
table underneath lists every VM with CPU, memory, disk, reboot state,
and uptime. This is the first dashboard I open in the morning.

### Proxmox VM resources

![Proxmox VM resources dashboard with CPU, memory, disk, and network panels](proxmox-vm-resources.png)

Per-VM CPU, memory, disk I/O, and network throughput at a glance.
Useful for catching an outlier before it starts affecting neighbours.

### Proxmox VM detail

![Proxmox VM detail dashboard with disk and network I/O, memory bars, and a summary table](proxmox-vm-detail.png)

The deeper cut: disk and network I/O timelines, actual memory used
per VM (both timeline and current bar chart), and a full summary
table with VMID, mode, status, tags, and uptime. If the overview
says something is off, this is where I look next.

### Kubernetes CoreDNS

![CoreDNS dashboard with request/response rates, cache size, and hit ratio](coredns.png)

Cluster DNS health: request rate, query-type breakdown, response
codes, duration percentiles, and cache hit ratio. A quiet CoreDNS
dashboard is an underrated signal that the cluster is behaving.

### Loki log explorer

![VM Log Explorer showing log volume by host and live log stream](loki-log-explorer.png)

The Loki-backed log explorer. Top: log volume per host so I can
see which VM is chatty at a glance. Bottom: live log stream with
host/log-type/app filters. Label extraction happens at the Vector
edge so filters are fast and consistent. See
[lessons-learned/elastic-to-loki-migration.md](../lessons-learned/elastic-to-loki-migration.md)
for why the stack looks the way it does.

### Ansible Runs dashboard (Prometheus-sourced)

![Ansible Runs Grafana dashboard with run status, duration, and per-playbook success history](ansible-runs-dashboard.png)

A Grafana dashboard fed by textfile-collector metrics emitted from
each Ansible run. Last-run status, duration, total playbooks
executed, playbook-level pass/fail history, and per-playbook
success timeline. Treats Ansible runs as a first-class observable
rather than fire-and-forget.

### ARA, Ansible Run Analysis

![ARA web UI listing recent playbook runs with status, duration, and host facts](ara-ansible-runs.png)

ARA is the "what actually happened" layer on top of Ansible. Every
playbook run lands here with per-task status, host facts, and
output, so post-mortems on a failed run do not require digging
through terminal history.

## Capture conventions

- Screenshots come from the live lab. Internal hostnames and RFC1918
  IPs are visible where Grafana shows them; that is intentional for
  a homelab portfolio.
- PNGs only. File sizes are kept sensible; the pre-commit
  `check-added-large-files` hook caps at 4096 KB.
- When recapturing, prefer Grafana's built-in "share panel" PNG
  export over OS-level screenshots so the crop stays consistent.
