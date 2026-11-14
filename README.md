# homelab-docs

Source of truth for my homelab: architecture, networking, services, configuration decisions, runbooks, and tested configurations.

## Layout

- `docs/architecture/` architecture design baselines and CTRoadmap exports
- `docs/networking/` network design and tested network configurations

## Architecture

- [Network architecture v0.4-simple](docs/architecture/network-v0.4-simple.md): **CURRENT TARGET / PLANNING**, the simplified design with four active VLANs
- [Network architecture v0.3](docs/architecture/network-v0.3.md): **FROZEN HISTORICAL DESIGN**, the previous, more segmented design

## Secrets policy

**Secrets must NEVER be committed to this repository.** This includes:

- private keys
- WireGuard configs (`*.conf`)
- passwords
- API tokens
- certificates
- any other credentials

This repository documents settings and behavior only. Store secret material separately in encrypted storage. The `.gitignore` blocks common secret file patterns, but it is a safety net and not a substitute for checking what you stage.
