# Media stack

Jellyfin, the *arr apps, and supporting services.

## What is here

- Library layout (Movies and TV Shows as separate typed libraries).
- Jellyfin GPU transcoding notes.
- *arr app conventions and NFS/SQLite constraints.

## Conventions

- Libraries are split by media type. Mixed libraries cause metadata and
  playback issues at scale; see
  `lessons-learned/jellyfin-library-split.md`.
- *arr apps are scaled 0 then 1 for restarts, never rolling-restarted.
  See `lessons-learned/arr-rollout-restart.md`.

TODO list every service and its role.
