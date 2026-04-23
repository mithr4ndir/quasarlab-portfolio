# Lessons learned

Post-incident writeups and design retrospectives. Each entry follows the
same template so they are easy to scan and compare.

## Index

- [alertmanager-spof.md](alertmanager-spof.md) single Alertmanager was a
  silent SPOF; fixed with HA + PDB, dead-man's-switch pending.
- [jellyfin-library-split.md](jellyfin-library-split.md) mixed media
  library split into typed Movies + TV Shows after a Jellyfin 10.11 upgrade.
- [hdr-flicker-hdmi.md](hdr-flicker-hdmi.md) G16 + 5K2K HDR flicker was
  not software; it was a marginal DP-to-USB-C cable.
- [grafana-k8s-migration.md](grafana-k8s-migration.md) Grafana VM moved
  into Kubernetes; Angular plugin CrashLoop follow-up.
- [arr-rollout-restart.md](arr-rollout-restart.md) why `kubectl rollout
  restart` kills *arr apps on NFS and what to do instead.
- [elastic-to-loki-migration.md](elastic-to-loki-migration.md) the
  flagship retrospective: why the lab moved off the Elastic Stack.

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
