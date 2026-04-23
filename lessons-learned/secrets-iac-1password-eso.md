# Secrets-as-code on 1Password, and what the rate limit taught us

## Date

Design: ongoing. First rate-limit incident: 2026-04-18. Recurrence root
cause identified and documented 2026-04-19. Hardening rollout started
2026-04-22.

## Impact

Two incidents where the 1Password Families account's per-account rate
limit got drained, and every downstream consumer (Ansible, External
Secrets Operator, any playbook that did a `op read`) failed until the
rolling window cleared. No data loss; substantial "nothing can deploy
right now" time.

## Duration

Incident windows measured in hours; design implications measured in
weeks of work to harden properly.

## Category

secrets management, rate limiting, Kubernetes, Ansible, supply chain,
reliability

## Summary

The lab treats secrets as code: no plaintext in git, no plaintext on
disk, everything flows from 1Password through External Secrets Operator
(for Kubernetes) and an Ansible lookup wrapper (for everything else).
This pattern is correct. It also has an operational foot-gun: the
1Password Service Account rate limit is **per account**, not per service
account, and anything that calls `op` freely will eventually run the
account dry. The fix is not "use 1Password less." The fix is defense in
depth: caching, a kill switch, a circuit breaker, an explicit ordering
rule for rollouts, and a cheap way to observe current quota state.

## Background

Before this design, a handful of secrets lived in Ansible Vault, others
in plain `tfvars`, others still in Kubernetes Secrets committed to git
from a private repo. This was adequate at one point. It stopped being
adequate once the lab grew to three clusters' worth of workloads and a
dozen VMs.

Goals for the redesign:

- Exactly one source of truth (1Password).
- No plaintext in any git repo, public or private.
- Declarative flow into both Kubernetes (ESO) and host configuration
  (Ansible playbooks).
- Auditable rotations.

## What happened, round one

On 2026-04-18, ESO started logging `rate limit exceeded` errors on a
roughly seven-minute cadence. Every ClusterSecretStore validation was
hitting the rate-limited API, and the retry storm was making the
problem worse.

The initial instinct was "ESO is hammering the API." True in effect,
false in cause.

The real source of the drain: two Ansible systemd timers on the
automation host, `ansible-proxmox.timer` and `ansible-security.timer`.
Each run called `op read` / `op item get` for vault items (PVE API
credentials, Wazuh credentials, and so on). Over the day those calls
chewed through the Families-tier quota. ESO was a victim, then an
aggravator; it was not the culprit.

Once the timers stopped, the `USED` counter stopped climbing and the
rolling 24-hour window recovered.

## Investigation

Key realisations:

- The 1Password rate limit is **per account**, not per service account.
  Creating a second service account does not buy you more quota.
- `op service-account ratelimit` is a **free** control-plane call. It
  reports `USED`/`LIMIT` without itself consuming quota. That makes it
  the right observability primitive.
- ESO failures were loud. Ansible timer failures were quiet. The loud
  one is often not the cause.
- A retry storm from a rate-limited client extends the window and
  masks the source. Stopping the storm is a prerequisite to
  diagnosing.

## Root cause

Multiple independent consumers calling `op read` with no coordination,
no caching, and no kill switch. Given enough time (and enough timers),
any of them could drain the account for all of them.

## Fix: defense in depth

### Layer 1: caching

`op_secret_cache` (Ansible-side): a file-backed cache with a 12-hour
TTL wrapping the 1Password lookup plugin. Playbooks read through
environment-variable lookups with an assert-fail-fast check so a
cache miss that cannot be satisfied stops the playbook instead of
quietly calling the API.

### Layer 2: kill switch

`op-killswitch.sh` (Ansible-side, PR #105 in `ansible-quasarlab`):
auto-trips on rate-limit errors, sets a 24-hour TTL, exposes a
Prometheus metric so the state is visible in Grafana and alertable
through Alertmanager. Any playbook that hits the switch fails fast
instead of extending the rate-limit window by retrying.

### Layer 3: scoped service accounts

The ESO service account is separate from the Ansible service account.
This does not buy more quota (same account underneath), but it does:

- isolate the audit trail ("which system drained this?");
- give us a granular rotation story (rotate one without rotating the
  other);
- enforce least privilege per-consumer.

### Layer 4: cheap observability

`op_ratelimit_collector` Ansible role (PR #121) samples
`op service-account ratelimit` every five minutes. Quota state lives
in Prometheus; Grafana plots it; alerts fire on high utilisation
before we hit the wall.

### Layer 5: structural ESO circuit breaker (in flight)

The op-killswitch protects **Ansible-side** `op` calls. ESO calls
1Password through its own code path and needs its own circuit
breaker. That is the Phase 0 work in umbrella
`mithr4ndir/k8s-argocd#147`, sub-issue #148, spec
`.spec-workflow/specs/eso-rate-limit-hardening/`.

### Layer 6: rollout ordering rule

Phase 2 of the secrets-in-IaC initiative adds roughly seven new
ExternalSecrets in the media namespace. Adding them before ESO has
circuit-breaker protection would re-create the 2026-04-18 failure
mode at higher blast radius. The ordering rule (documented in the
umbrella issue and in the team's memory): **do not open a PR for
Phase 2 tasks until the Phase 0 circuit breaker is merged and has
been running in production for at least one full refresh window
(24 hours).** Phase 1 (Ansible side) can run in parallel with Phase 0.

## Takeaways

- "Use a secret manager" is the right architectural choice. It is not
  a finished project; it is the start of a reliability story.
- Know the rate-limit scope before you design the quota story. "Per
  service account" and "per account" have very different implications
  for horizontal scaling of consumers.
- Free control-plane calls are gold. Build your observability around
  them so you can see quota state without paying quota to see it.
- The loud failure is not always the cause. ESO screaming on a
  7-minute cadence was a symptom; timers in another repo were
  draining the account.
- Kill switches beat retries for rate-limited clients. A client that
  stops after one error protects the whole system; a client that
  retries makes the problem longer.
- When a protection lands on side A of a system (Ansible), ship the
  equivalent on side B (ESO) before you expand scope on side B.
  Write the ordering down so future-you does not get clever.
- Capture the "how to apply next time" as a runbook the same day you
  figure it out. Memory decays fast; the next drain is always
  somewhere else on the schedule.

## How to triage the next rate-limit incident

1. Confirm the state cheaply: `op service-account ratelimit` (free).
2. Stop active drains: `systemctl stop ansible-proxmox.timer
   ansible-security.timer` on the automation host. If the `USED`
   counter stops climbing, timers were the culprit.
3. Scale ESO to zero to stop the retry storm:
   `kubectl scale deploy external-secrets -n external-secrets
   --replicas=0`. ESO is a symptom amplifier, not (usually) the
   cause; scaling to zero is a noise-reduction step, not a fix.
4. Do **not** run any playbook that pulls secrets during a rate-limit
   state. Failing calls extend the window.
5. Wait for the 24-hour rolling window. Use the quota dashboard to
   see progress.
6. When clear, scale ESO back up:
   `kubectl scale deploy external-secrets -n external-secrets
   --replicas=1`.

## Follow-ups

- [x] Caching (`op_secret_cache`, Ansible-side).
- [x] Kill switch (`op-killswitch.sh`, Ansible-side).
- [x] Scoped service accounts.
- [x] `op_ratelimit_collector` for Prometheus.
- [ ] ESO circuit breaker (Phase 0 of umbrella #147). Must land before
      media namespace secrets expansion.
- [ ] Proxmox dynamic inventory: the inventory plugin still calls `op`
      per fork, outside the op-secret-cache wrapper, burning reads on
      each `vm_baseline.yml` run. Root-cause fix is to vault the
      Proxmox API token; interim mitigation is documented.
- [ ] Quarterly fire-drill: exhaust the quota on purpose in a
      pre-production window to verify the kill switch and the ESO
      circuit breaker behave as designed.
