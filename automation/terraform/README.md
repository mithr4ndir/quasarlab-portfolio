# Terraform

Notes on the Terraform toolchain used to manage the lab.

## What is here

- Provider layout (Proxmox, Cloudflare, etc).
- State backend and locking strategy.
- Module conventions.

## Conventions

- State is remote and encrypted. Locking is mandatory.
- Sensitive variables are marked `sensitive = true` and sourced from
  secret backends, never from `*.tfvars` in git.
- `tfsec` / `checkov` in CI for every PR.

TODO link representative modules and the state backend config (scrubbed).
