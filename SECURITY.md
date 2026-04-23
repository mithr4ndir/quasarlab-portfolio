# Security Policy

## Reporting a vulnerability

Please do not open public issues for security reports. Use
[GitHub Security Advisories](https://github.com/mithr4ndir/quasarlab-portfolio/security/advisories/new)
to file a private report.

You can expect:

- An acknowledgement within 72 hours.
- A triage note within 7 days describing severity and next steps.
- Credit in the advisory on request.

## Scope

In scope:

- The contents of this repository: documentation, example configuration,
  CI workflows, and scripts shipped in this repo.
- Sanitization failures (for example, a real hostname, IP, secret, or token
  that leaked into a commit).

Out of scope:

- Infrastructure that is not published here. The repo is documentation; the
  live lab is not internet-exposed by default.
- Rate-limit or denial-of-service reports against third-party services we do
  not operate.

## Secret hygiene

All commits run through:

- [gitleaks](https://github.com/gitleaks/gitleaks) in pre-commit and CI
- [pre-commit](https://pre-commit.com/) hooks for trailing whitespace,
  end-of-file, YAML/JSON syntax, private keys, and large files

Any example values (tokens, IPs, hostnames) in this repo are synthetic. If
you find one that is not, treat it as a P1 and report privately.
