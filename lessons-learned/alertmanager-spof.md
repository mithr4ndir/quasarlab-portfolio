# Alertmanager single point of failure

## Date

Incident: 2026-04-05. Fix landed: 2026-04-06.

## Impact

The command-center VM went down for roughly six hours and not a single
alert was delivered. The alert path was quietly off during exactly the
window where it was most needed.

Prometheus counted 6,770 dropped notifications across the outage.

## Duration

Detection-to-resolution for the VM outage itself: six hours. Detection
of the real problem (Alertmanager was a SPOF) took longer because the
symptom was "silence", which is the same symptom as "nothing is wrong".

## Category

observability, high-availability, alerting

## Summary

A single-replica Alertmanager co-located on one Kubernetes node died when
that node crashed. Prometheus kept scraping and evaluating rules, but
had nowhere to send alerts. The fix was HA Alertmanager (two replicas
with soft pod anti-affinity and a PodDisruptionBudget) plus a new
`PrometheusNotificationsDropped` alert so the next silence cannot be
silent. A dead-man's-switch against an external probe landed in a
follow-up wave.

## Background

Alertmanager ran as a single replica inside `kube-prometheus-stack`.
The replica happened to be scheduled on one specific control-plane
node. That node had a separate latent issue (an etcd timeout cascade)
that would occasionally cause a hard crash.

Prometheus was healthy. Rules were evaluating correctly. The problem
was purely on the notification edge.

## What happened

The command-center VM lost network. Prometheus noticed almost
immediately and began raising alerts. Those alerts piled up in
Prometheus's notification queue with nowhere to go, because the
Alertmanager pod had gone down with its node.

From outside, everything looked fine. The Discord channel was quiet.
Grafana was quiet. The phone was quiet.

## Investigation

Post-incident, the investigation had three questions:

1. Why didn't we know the VM was down? Because Alertmanager was down.
2. Why didn't we know Alertmanager was down? Because there was no
   external watchdog; the only monitor for Alertmanager was
   Alertmanager.
3. Why did Alertmanager die? Because it was a single replica on a
   node that hard-crashed.

Each answer unblocked the next. (1) and (3) are straightforward
availability problems. (2) is a design flaw: "I would notice if alerts
stopped" is not a monitoring strategy.

## Root cause

Single-replica Alertmanager with no external probe of the notification
path. The specific triggering event (etcd timeout cascade on a single
node) made the pod die; the fact that losing one pod killed the whole
alert path is the actual root cause.

## Fix

Shipped immediately:

- Scaled Alertmanager to two replicas with soft `podAntiAffinity` so the
  scheduler prefers to spread them across nodes.
- Added a `PodDisruptionBudget` with `minAvailable: 1` so node drains,
  cordons, and voluntary disruptions cannot take the alert path to
  zero.
- Added a `PrometheusNotificationsDropped` alert (critical severity) so
  a stuck notification queue is itself a pageable event.

Shipped in a follow-up wave:

- External dead-man's-switch via a public heartbeat endpoint
  (Healthchecks.io). Prometheus pings it on a schedule; if the ping
  stops arriving, Healthchecks.io raises its own alert, completely
  independent of the cluster it is monitoring.
- Discord alert proxy made HA, baked to an immutable image, and wired
  into Dependabot plus Trivy image scanning in CI so the proxy itself
  is not the new SPOF.

## Takeaways

- Any component on the alert path is either HA or has a compensating
  external monitor. There is no third option.
- "We would notice" is not a monitoring strategy. Prove it with a
  dead-man's-switch that runs outside the thing it watches.
- Same-service monitoring of the alert path is circular. The monitor
  must live somewhere the watched thing cannot drag down with it.
- PDBs are not optional for HA workloads. Two replicas without a PDB
  still hit zero during a drain.
- Pod anti-affinity as `preferred` is the right default for a two-node
  workload on a three-node cluster. `required` would break scheduling
  during maintenance; `preferred` gives the scheduler room.

## Follow-ups

- [x] HA Alertmanager with PDB.
- [x] `PrometheusNotificationsDropped` alert.
- [x] External heartbeat / dead-man's-switch (Healthchecks.io).
- [x] Discord alert proxy made HA, image baked, CI scanning in place.
- [ ] Address the node-level etcd timeout cascade that caused the
      underlying crash. The alert path is protected now; the
      underlying instability deserves its own incident writeup.
