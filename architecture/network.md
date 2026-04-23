# Network architecture

## TL;DR

- Two edges. **Nginx Proxy Manager** fronts internal services on the
  LAN. **Cloudflare Tunnel** fronts the narrow set of services that
  need to be reachable from the public internet, with no open ports
  on the firewall.
- **MetalLB** assigns stable LoadBalancer IPs from a reserved pool to
  every user-facing Kubernetes service (Grafana, Alertmanager,
  Prometheus, Loki, Homepage, the Discord alert proxy).
- **kube-vip** provides the highly-available Kubernetes API endpoint
  so `kubectl` and ArgoCD can talk to the cluster through a single
  address even when one control-plane node is down.
- Segmentation is conceptual: management, services, and storage
  traffic are logically separated even on shared physical gear.
- No real IPs or VLAN tags appear in this repo. The lab is not
  internet-exposed by default; Cloudflare Tunnel is the one narrow
  path and it terminates at a dedicated, read-only surface.

## Deep dive

### Physical and logical layout

The lab sits behind a small number of switches and a single router.
Physically it is not exotic: a Proxmox HA pair, a TrueNAS box, and a
handful of IoT and workstation devices on the same Ethernet fabric.
Logically there are distinct roles for the traffic on that fabric:

- **Management.** Hypervisor UIs, IPMI/iDRAC where present, cluster
  control-plane traffic. Never exposed beyond the LAN.
- **Services.** Most pod-to-pod traffic and the LoadBalancer pool for
  user-facing Kubernetes services. This is where Grafana, Prometheus,
  Loki, Alertmanager, and the Homepage dashboard live from a client
  perspective.
- **Storage.** NFS exports from TrueNAS to Kubernetes worker nodes and
  to Proxmox guests. High-volume, latency-sensitive, and kept off the
  general-services path.
- **IoT and guest.** Isolated from the lab; can reach the internet but
  not the management or services path.

The public portfolio treats the specific VLAN tags and subnets as
sensitive. The shape is documented here; the numbers live in the
private DR repo.

### Kubernetes ingress path

A request from a LAN client to a Kubernetes service follows a short
path:

1. DNS resolution to the Nginx Proxy Manager address for the intended
   hostname (for example, `grafana.lab.internal`).
2. NPM terminates TLS with an internal Let's Encrypt cert and proxies
   upstream.
3. The upstream is a **MetalLB-assigned LoadBalancer IP**, not a
   cluster-internal Service. That means NPM talks to a stable address
   that persists across pod reschedules.
4. kube-proxy forwards the traffic to the right pod on the right
   node.

MetalLB runs in L2 mode with a reserved pool. Every externally
reachable Kubernetes service gets a predictable IP from that pool.
This is the pattern that lets me decommission a VM-based service
(old Grafana on a hypervisor VM) and replace it with a Kubernetes
one without any downstream client change: NPM upstream moves from the
VM IP to the MetalLB IP, everything else stays the same.

### Public surface: Cloudflare Tunnel

NPM is deliberately internal-only. The lab has zero inbound port
forwards. The one way anything in the lab reaches the public
internet is through a **Cloudflare Tunnel**:

- `cloudflared` runs as a Kubernetes Deployment in its own
  namespace.
- The tunnel token is mounted as a Kubernetes Secret sourced from
  1Password through External Secrets Operator. Rotating the token
  is a 1Password update plus an ESO sync, not a code change.
- The tunnel routes a single hostname
  (`showcase.<public-domain>`) to a dedicated read-only Grafana
  instance. No other routes. No catch-all. No admin paths. The full
  design lives in
  [`../docs/security/public-showcase.md`](../docs/security/public-showcase.md).
- Cloudflare edge does WAF, Bot Fight Mode, rate limiting, and
  HSTS. The tunnel is the only crossing; if it goes down, the
  public surface goes dark and nothing in the lab is reachable
  from outside. That is the intended failure mode.

### Observability traffic

Monitoring traffic rides the services network:

- **Prometheus** scrapes in-cluster ServiceMonitors and
  `file_sd`-driven static targets (the VM fleet, Proxmox
  exporters, TrueNAS). Scrape intervals are predictable; there is
  no push.
- **Vector** does the only push traffic: Kubernetes container logs
  from the DaemonSet, and journald plus `/var/log/**/*.log` from
  VM agents. Everything flows to Loki on the services network.
- **Alertmanager** sends outbound to Discord (via the in-cluster
  `discord-alert-proxy`) and to Healthchecks.io for the
  dead-man's-switch heartbeat. Those are the only egress paths from
  the alert pipeline, and both are explicit.
- **Wazuh agents** on every Linux host ship events to the Wazuh
  manager VM over its own TLS-secured channel. Wazuh traffic
  deliberately does not cross into the Loki pipeline: SIEM is its
  own discipline.

### Edge rules (Cloudflare side)

- Managed WAF rules plus a custom rule set that denies every
  Grafana admin path (`/api/admin/`, `/logout`, `/dashboards/new`,
  plugin-install endpoints). Only `/public-dashboards/` is allowed
  through.
- Per-IP rate limits on all paths, stricter limits on any
  datasource-proxy path.
- HTTPS only, minimum TLS 1.2, preferred 1.3, HSTS preload.
- Bot Fight Mode on. Browser integrity check on.

### Why two edges

A common question: why NPM at all if Cloudflare Tunnel can reach
in-cluster services?

Because the two edges serve different audiences. NPM handles the
large surface of internal services that I and my household use
constantly: Homepage, Sonarr/Radarr/Jellyfin, NZBGet, Grafana,
Proxmox UIs, TrueNAS UI. It is TLS-terminating, LAN-local, zero
authentication beyond the services' own. Putting any of that behind
Cloudflare would be over-architecting.

Cloudflare Tunnel handles one surface: a read-only public Grafana,
and any future narrowly-scoped public endpoint. Different blast
radius, different security posture, different tooling.

### Related code

- [`k8s-argocd`](https://github.com/mithr4ndir/k8s-argocd) holds
  the MetalLB and kube-vip configuration, NetworkPolicy resources,
  ingress definitions, and the (future) `cloudflared` Deployment.
- [`ansible-quasarlab`](https://github.com/mithr4ndir/ansible-quasarlab)
  handles Nginx Proxy Manager upstream updates when LoadBalancer
  IPs move (rare, but happens, e.g. the Grafana VM-to-K8s
  migration).
- [`terraform-quasarlab`](https://github.com/mithr4ndir/terraform-quasarlab)
  manages the Cloudflare zone records.
