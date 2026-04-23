# Jellyfin library split and dedup

## Date

Resolved 2026-04-17

## Impact

TODO (degraded metadata, duplicate entries, transcoding oddities)

## Duration

TODO

## Category

media, jellyfin

## Summary

After upgrading Jellyfin to 10.11, a mixed-type library (Movies and TV
together) started surfacing duplicate and mis-typed entries. The fix
was to split into two typed libraries, dedup, and preserve per-user
UserData across the rebuild.

## Background

TODO describe why the library was originally mixed and what changed in
10.11 that surfaced the issue.

## What happened

TODO

## Investigation

TODO

## Root cause

TODO

## Fix

- Split the mixed library into a Movies library and a TV Shows library,
  each with the correct content type.
- Ran a dedup pass on the filesystem side before Jellyfin re-indexed.
- Preserved UserData (watch progress, favourites, play counts) across
  the rebuild using the Jellyfin 10.11 auth header + UserData-preservation
  trick. TODO capture the exact API calls.

## Takeaways

- Never run a mixed-type Jellyfin library at scale. Typed libraries are
  cheap and the upgrade behaviour is predictable.
- Watch-progress data is the thing users actually care about. Any
  library rebuild plan that loses it is not a plan.

## Follow-ups

- [ ] Document the UserData-preservation procedure as a reusable runbook.
- [ ] Add a dashboard panel for "libraries by content type" so any
      future mixed library is visible at a glance.
