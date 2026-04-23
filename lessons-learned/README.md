# Lessons learned

Post-incident writeups and design retrospectives. Each entry follows
the same template so they are easy to scan and compare. The campaign
log, if you like.

## Index

Flagship retrospectives (filled in, ready to read):

- [elastic-to-loki-migration.md](elastic-to-loki-migration.md) ![status](https://img.shields.io/badge/status-resolved-brightgreen)
  the flagship retrospective: why the lab moved off the Elastic Stack
  to Loki + Vector + Wazuh.
- [pve-ha-outage.md](pve-ha-outage.md) ![status](https://img.shields.io/badge/status-resolved-brightgreen) ![followups](https://img.shields.io/badge/follow--ups-open-blue)
  2026-04-13 HA fence, nine hours of silent VM outage, four-day
  multi-repo remediation.
- [secrets-iac-1password-eso.md](secrets-iac-1password-eso.md) ![status](https://img.shields.io/badge/design-in--flight-yellow)
  how the lab does secrets-as-code on 1Password, and the defense-in-depth
  added after two rate-limit incidents.
- [alertmanager-spof.md](alertmanager-spof.md) ![status](https://img.shields.io/badge/status-resolved-brightgreen)
  single Alertmanager was a silent SPOF. HA + PDB + external
  dead-man's-switch.
- [grafana-k8s-migration.md](grafana-k8s-migration.md) ![status](https://img.shields.io/badge/status-resolved-brightgreen) ![followups](https://img.shields.io/badge/follow--ups-open-blue)
  Grafana moved from VM to Kubernetes, SQLite to PostgreSQL via
  pgloader, old VM decommissioned by the full checklist.
- [arr-rollout-restart.md](arr-rollout-restart.md) ![rule](https://img.shields.io/badge/rule-in--effect-purple)
  why `kubectl rollout restart` kills *arr apps on NFS and what to do
  instead.

Shorter retrospectives (partial, need more detail from the user):

- [jellyfin-library-split.md](jellyfin-library-split.md) ![status](https://img.shields.io/badge/status-stub-lightgrey)
  mixed media library split into typed Movies + TV Shows after a
  Jellyfin 10.11 upgrade.
- [hdr-flicker-hdmi.md](hdr-flicker-hdmi.md) ![status](https://img.shields.io/badge/status-stub-lightgrey)
  G16 + 5K2K HDR flicker was not software; it was a marginal
  DP-to-USB-C cable.

## Template

Each lesson uses these sections:

- Date
- Impact
- Duration
- Category
- Summary
- Background
- What happened
- Investigation
- Root cause
- Fix
- Takeaways
- Follow-ups
