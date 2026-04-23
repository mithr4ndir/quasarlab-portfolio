# Alertmanager single point of failure

## Date

TODO

## Impact

TODO (who was paged, what silence gap opened up)

## Duration

TODO

## Category

observability, high-availability

## Summary

A single-replica Alertmanager was the silent SPOF in the alert path. The
fix was to run two replicas with a PodDisruptionBudget. A dead-man's-switch
(Watchdog alert plus external probe) is still pending.

## Background

TODO describe the pre-incident Alertmanager topology, how alerts flowed
from Prometheus to Discord, and why the SPOF was not obvious.

## What happened

TODO describe the trigger event and the observed symptoms.

## Investigation

TODO

## Root cause

TODO

## Fix

- Scaled Alertmanager to 2 replicas with peer gossip.
- Added a PodDisruptionBudget (`minAvailable: 1`) so node drains cannot
  take the alert path to zero.
- TODO capture the exact Helm values diff.

## Takeaways

- Any component on the alert path needs HA or a compensating monitor.
- "We would notice" is not a monitoring strategy. Prove it with a dead
  man's switch.

## Follow-ups

- [ ] Land the Watchdog alert + external probe as the dead-man's-switch.
- [ ] Add a runbook entry for "alerts silent, is the pager actually
      alive?"
