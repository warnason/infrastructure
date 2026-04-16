# Infrastructure

Self-hosted development and deployment infrastructure for personal technical projects.

Runs a private Git forge (Forgejo), an automated reverse proxy with TLS (Caddy), and serves prototype applications under development.

## Overview

- **Host:** Ubuntu 24.04 LTS on dedicated hardware (Hetzner, Falkenstein)
- **Storage:** Software RAID 1 for the operating system, separate disk for non-critical data
- **Orchestration:** Docker Compose
- **Reverse proxy:** Caddy 2, with automatic Let's Encrypt certificates
- **Git hosting:** Forgejo with PostgreSQL backend
- **Firewall:** UFW, restricting inbound traffic to SSH, HTTP, HTTPS, and Git-over-SSH
- **Intrusion prevention:** fail2ban on SSH

## Services

| Service | URL                      | Purpose                        |
|---------|--------------------------|--------------------------------|
| Forgejo | https://git.stifting.at  | Self-hosted Git and CI/CD      |
| App     | https://bim.stifting.at  | Currently hosted prototype     |

## Repository layout

```
.
├── caddy/                 # Reverse proxy configuration
│   ├── docker-compose.yml
│   └── Caddyfile
├── forgejo/               # Git server configuration
│   ├── docker-compose.yml
│   └── .env.example
├── docs/                  # Setup notes and runbooks
│   └── setup.md
└── README.md
```

## Bootstrapping a new host

See [docs/setup.md](docs/setup.md) for the full installation procedure.

High-level steps:

1. Provision Ubuntu 24.04 via `installimage` with RAID 1
2. Harden SSH, create an admin user, enable UFW and fail2ban
3. Install Docker and create the shared `web` network
4. Deploy Caddy, then Forgejo
5. Configure DNS records for the relevant subdomains

## Security notes

- SSH uses key-only authentication; root login is disabled
- Secrets (`.env`, passwords) are never committed; templates are provided as `.env.example`
- Unattended security upgrades are enabled
- TLS certificates are issued and renewed automatically via ACME

## License

MIT
