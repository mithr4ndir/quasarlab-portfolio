# Grafana VM to Kubernetes migration

## Date

TODO

## Impact

TODO

## Duration

TODO

## Category

observability, migration

## Summary

Grafana was moved off a standalone VM and into the Kubernetes cluster as
a proper Helm-managed release. The old VM was destroyed per the
decommissioning checklist. One follow-up remains: an Angular-based
plugin is in CrashLoop and needs replacement or removal.

## Background

TODO describe the VM-based Grafana setup and why the move was worth doing.

## What happened

TODO migration steps, cutover, and the data that came across (dashboards,
datasources, users).

## Investigation

TODO

## Root cause

N/A (planned migration, not an incident)

## Fix

- Grafana deployed via Helm in Kubernetes with persistent storage and
  ESO-mounted secrets.
- Dashboards provisioned from `observability-quasarlab`.
- Old VM destroyed after verifying:
  - Prometheus static targets removed.
  - Ansible inventory entries removed.
  - Terraform state cleaned.
  - Alertmanager routes unaffected.
  - DNS and NPM upstreams updated.

## Takeaways

- Decommissioning is its own checklist. Destroying a VM is step N of
  many, not the whole job.
- Angular-based Grafana plugins are a migration hazard in modern
  Grafana builds.

## Follow-ups

- [ ] Replace or remove the CrashLooping Angular plugin.
- [ ] Verify Grafana pod PDB and HA story.
