# Screenshots

Static visual evidence for the portfolio. Safe to commit, always
available, no infrastructure dependency.

## What is here

- Grafana dashboard PNGs sourced from the sanitised showcase instance.
- Topology and architecture diagrams (no real IPs or hostnames).
- Before/after captures for lessons-learned entries.

## Conventions

- Redact hostnames, IPs, and user data before committing. If in doubt,
  redact.
- Prefer PNG over JPEG for UI screenshots.
- Keep file sizes sensible; the pre-commit `check-added-large-files`
  hook caps at 4096 KB. Hero-class images may run larger, but most
  dashboard captures should stay well under 1 MB.

TODO add the first set of captures.
