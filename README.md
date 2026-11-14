# homelab-docs

Source of truth for my homelab: architecture, networking, services, configuration decisions, runbooks, and tested configurations.

## Layout

- `docs/networking/` network design and tested network configurations

## Secrets policy

**Secrets must NEVER be committed to this repository.** This includes:

- private keys
- WireGuard configs (`*.conf`)
- passwords
- API tokens
- certificates
- any other credentials

This repository documents settings and behavior only. Store secret material separately in encrypted storage. The `.gitignore` blocks common secret file patterns, but it is a safety net and not a substitute for checking what you stage.
