# Eirdom deployment baseline

## Current placement

- `eirdom-docker-01`: Docker application platform on Ubuntu 26.04 LTS, address `10.1.50.10`.
- Eirdom Intelligence: separate AI-capable system; not deployed on the Docker host.
- Active Directory and PKI: separate Windows Server systems.
- Traefik is the only application ingress on ports 80/443.
- Cloudflare Tunnel publishes only approved external applications and targets a local origin—not the public hostname itself.

## Required gates before adding another service

- CPU temperatures are within safe operating range.
- Existing containers are healthy or have a documented health-check exception.
- Backups are current and restore-tested.
- Compose configuration renders cleanly.
- State directories have verified ownership.
- Local Traefik routing works before a Cloudflare route is added.
- Secrets are stored outside Git.

## NetBox go-live gate

Do not start NetBox until PostgreSQL and Valkey storage permissions, LDAP over TLS, plugin compatibility, backup coverage, and host thermal stability are verified.
