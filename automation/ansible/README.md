# Ansible

Notes on the Ansible toolchain used to manage the lab.

## What is here

- Inventory model (dynamic from the Proxmox API).
- Playbook conventions and role layout.
- 1Password integration patterns and their gotchas.

## Conventions

- Secrets never live in plain group_vars/host_vars. 1Password first,
  vaulted files as a fallback.
- Dynamic inventory runs behind a caching wrapper where possible to
  avoid hammering the 1Password rate limit.

TODO link representative playbooks and the op-secret-cache pattern.
