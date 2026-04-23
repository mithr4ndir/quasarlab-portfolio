# Public showcase design

A read-only public Grafana instance, published behind a Cloudflare Tunnel,
so visitors (recruiters, peers, the curious) can see live-ish dashboards
without the lab being internet-exposed.

This document is the design. Nothing here is speculative; it describes
the target architecture the repo will converge to.

## Goals

- Let anonymous visitors view a small curated set of dashboards.
- Never expose the real Grafana instance, Prometheus, Loki, or any
  infrastructure directly to the public internet.
- Leak no hostnames, IPs, topology, or alert content.
- Fail safe: if the tunnel or showcase instance dies, the lab is unaffected.

## Non-goals

- Letting anyone log in, explore freely, or run arbitrary queries.
- Publishing alert state, silences, or the runbook links baked into alerts.
- Supporting write paths of any kind.

## Topology

Two Grafana instances. They are not the same pod, not the same namespace,
not the same values file, not the same datasources.

```
Internet
   |
   v
Cloudflare edge (WAF, Bot Fight, rate limiting, HTTPS only)
   |
   v
Cloudflare Tunnel
   |
   v
cloudflared Deployment (namespace: cloudflared)
   |
   v
grafana-showcase Service (namespace: grafana-showcase)
   |
   v
Prometheus read-only pre-aggregated datasource
```

The existing internal Grafana stays where it is, reachable only through
Nginx Proxy Manager on the internal network. NPM is not in the public
path at all. The tunnel bypasses NPM entirely for the showcase hostname.

## Tier strategy

- **Tier 1, static.** Grafana dashboard snapshots and PNG screenshots
  committed to this repo under `screenshots/` and `dashboards/`. Always
  safe, always available, no infrastructure dependency.
- **Tier 2, live.** Grafana's built-in
  [public dashboards](https://grafana.com/docs/grafana/latest/dashboards/dashboard-public/)
  feature on the showcase instance, reached via the tunnel. This is the
  recommended tier for live content.

Tier 1 is the floor. Tier 2 is opt-in and can be disabled at any time
without breaking the portfolio.

## Grafana showcase instance

Separate Helm release, separate namespace. Hardened `grafana.ini`:

- `auth.anonymous.enabled = true`
- `auth.anonymous.org_role = Viewer`
- `auth.disable_login_form = true`
- `auth.disable_signout_menu = true`
- `users.allow_sign_up = false`
- `users.viewers_can_edit = false`
- `explore.enabled = false`
- `alerting.enabled = false`
- `unified_alerting.enabled = false`
- `panels.disable_sanitize_html = false`
- `security.cookie_secure = true`
- `security.cookie_samesite = strict`
- `security.strict_transport_security = true`
- `snapshots.external_enabled = false`

Only the public-dashboards feature is exposed. Admin UI is unreachable
because anonymous users cannot see it and there is no login form.

## Datasource

A dedicated Prometheus datasource that reads only pre-aggregated metrics
(recording rules), not raw time series. Labels are scrubbed of hostnames,
IPs, and any topology detail. Concretely:

- Recording rules aggregate to `instance_role` style labels
  (`compute`, `storage`, `control-plane`) instead of real hostnames.
- No pod names, no namespaces that hint at tenants, no container IDs.
- No Loki datasource attached to the showcase. Log bodies are the
  highest-leak surface; they stay internal.

## Cloudflare Tunnel

- `cloudflared` runs as a Kubernetes Deployment in namespace
  `cloudflared`, with the tunnel token mounted from 1Password via ESO.
- The tunnel routes exactly one hostname (`showcase.<public-domain>`) to
  the `grafana-showcase` Service inside the cluster.
- No other routes. No SSH, no admin, no catch-all.
- The tunnel Deployment has `NetworkPolicy` restricting egress to
  Cloudflare edges and the showcase Service only.

## Cloudflare edge controls

- WAF: managed rules + custom rule set blocking `/api/`, `/admin/`,
  `/login`, `/logout`, `/dashboards/new`, and the Grafana plugin install
  paths. Public dashboards live under `/public-dashboards/`; everything
  else is denied at the edge.
- Bot Fight Mode: on.
- Rate limiting: per-IP cap on all paths, stricter cap on
  `/api/datasources/proxy/*`.
- HTTPS only, HSTS with `max-age=31536000; includeSubDomains; preload`.
- Minimum TLS 1.2. Prefer 1.3.
- Browser integrity check: on.

## Monitoring the showcase

The showcase has its own uptime check and its own Alertmanager route
(quiet channel, not the main Discord). Noise from the public internet
should not wake anyone up, but a tunnel that has been offline for an
hour is worth knowing about.

## Failure modes and responses

- Tunnel token leaked: rotate in Cloudflare, update 1Password, ESO
  propagates within one refresh cycle. No code change needed.
- Public dashboard accidentally exposes a sensitive label: revoke the
  public dashboard in Grafana, fix the recording rule, re-publish.
- Abuse traffic at the edge: tighten the WAF rule set, drop the
  public-dashboards path for the offending pattern, Cloudflare absorbs
  the flood.
- Showcase instance compromised: it has no write access, no secrets
  beyond a scoped datasource token, and is network-isolated from the
  rest of the cluster. Blast radius is the showcase namespace.

## Rollout checklist

- [ ] Create `grafana-showcase` Helm release with hardened `grafana.ini`.
- [ ] Create scrubbed recording-rule set in `observability-quasarlab`.
- [ ] Attach the scrubbed Prometheus datasource, verify no raw labels leak.
- [ ] Deploy `cloudflared` with NetworkPolicy and ESO-mounted token.
- [ ] Configure Cloudflare WAF rules, rate limits, HSTS.
- [ ] Publish one curated dashboard via the public-dashboards feature.
- [ ] Add uptime probe and quiet Alertmanager route.
- [ ] Link the public URL from the top-level `README.md`.
