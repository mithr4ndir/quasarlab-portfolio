# Architecture

High-level design notes and diagrams for the quasarlab homelab. Each file
covers one axis of the system. Diagrams are intentionally abstract: real
hostnames, IPs, and topology are never committed here.

## Contents

- [network.md](network.md) physical and logical network layout
- [kubernetes.md](kubernetes.md) cluster topology, control plane HA, ingress
- [observability.md](observability.md) metrics, logs, alerts, and dashboards
- [data.md](data.md) storage layout, backup policy, disaster recovery

## Conventions

- No real IPs or hostnames. Use `example.internal`, `10.0.0.0/8` only in the
  most abstract sense, or just "the ingress VIP".
- No hardware serial numbers, MAC addresses, or cloud account IDs.
- When a detail is sensitive, describe the pattern instead of the value.
