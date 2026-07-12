# Eirdom — Master Services Reference
> Last Updated: April 2026

---

## Overview

This document is the single source of truth for all services running
across the Eirdom infrastructure. It covers Docker containers, VMs,
UniFi network devices, and supporting services.

### DNS Architecture

All services use **split-horizon DNS** on a single `*.eirdom.homes`
subdomain pattern.

- **Internal clients** resolve `*.eirdom.homes` to `10.1.50.10`
  (Traefik) via the AD DNS split-horizon zone on EIRDOM-DC-01
- **External clients** resolve `*.eirdom.homes` to Cloudflare proxy
  IPs via public Cloudflare DNS
- **Cloudflare Tunnel** only exposes services with an active ingress
  rule in Zero Trust — all other subdomains exist in DNS but are
  unreachable from the internet

Services marked **Internal Only** have a Cloudflare DNS record but
no tunnel ingress rule — they are unresolvable from the public
internet even though the subdomain exists.

---

## Service Status Legend

| Symbol | Meaning |
|--------|---------|
| 🟢 | Production |
| 🟡 | Planned / Not yet deployed |
| 🔴 | Offline / Decommissioned |

---

## Network Infrastructure

### UniFi Devices

| Device | Hostname | IP | VLAN | URL | Credentials |
|--------|----------|----|------|-----|-------------|
| UDM-Pro-Max | `udm-pro-max` | 10.1.1.1 | 1 | `https://10.1.1.1` | Password manager |
| USW-Pro-Max-48-PoE (Core) | `usw-core-01` | 10.1.1.2 | 1 | Managed via UDM | — |
| USW-Pro-Max-48-PoE (Distribution) | `usw-dist-01` | 10.1.1.3 | 1 | Managed via UDM | — |

> UniFi switches are managed exclusively through the UniFi Network
> application on the UDM-Pro-Max. Direct switch web UI access is not
> enabled. SSH access to the UDM is restricted to `obj_my_device` only
> — see `lan-rules.md`.

---

## Physical Servers

| Server | Hostname | IP | VLAN | OS | Role |
|--------|----------|----|------|----|------|
| 🟢 Proxmox Host | `EIRDOM-PVE-01` | 10.1.10.5 | 10 | Proxmox VE 8.x | Primary hypervisor |
| 🟢 Docker Host | `EIRDOM-DOCKER-01` | 10.1.50.10 | 50 | Ubuntu 26.04 LTS | All Docker services |

---

## Virtual Machines

| Status | VM | Hostname | IP | VLAN | OS | Role |
|--------|-----|----------|----|------|----|------|
| 🟢 | VM 100 | `EIRDOM-DC-01` | 10.1.10.10 | 10 | Windows Server 2025 | AD DS, DNS, DHCP, NPS |
| 🟢 | VM 101 | `EIRDOM-RCA-01` | Air-gapped | None | Windows Server 2025 | Offline Root CA |
| 🟢 | VM 102 | `EIRDOM-SUB-01` | 10.1.10.12 | 10 | Windows Server 2025 | Issuing / Subordinate CA |
| 🟢 | VM 120 | `EIRDOM-WAZUH-01` | 10.1.60.10 | 60 | Ubuntu 26.04 LTS | Wazuh Manager + Indexer + Dashboard |
| 🟢 | VM 130 | `EIRDOM-SONION-01` | 10.1.60.20 | 60 | Oracle Linux 9 | Security Onion 3.0 Standalone |
| 🟢 | VM 140 | `EIRDOM-HAOS-01` | 10.1.20.10 | 20 | Home Assistant OS | Smart home hub |

### VM Management URLs

| Service | Internal URL | External URL | Notes |
|---------|-------------|-------------|-------|
| Proxmox Web UI | `https://proxmox.eirdom.homes` | Internal only | Port 8006 — Traefik reverse proxies |
| Wazuh Dashboard | `https://wazuh.eirdom.homes` | Internal only | Port 443 via Traefik |
| Security Onion | `https://securityonion.eirdom.homes` | Internal only | Port 443 via Traefik |
| Home Assistant | `https://homeassistant.eirdom.homes` | Internal only | Port 8123 via Traefik |

---

## Docker Services

All Docker services run on `EIRDOM-DOCKER-01` (10.1.50.10) on VLAN 50.
All external access routes through Traefik at 10.1.50.10:443 then
onward through the Cloudflare Tunnel. No container ports are exposed
directly to any VLAN.

---

### Traefik — Reverse Proxy

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `traefik` |
| Image | `traefik:v3.3` |
| VLAN | 50 |
| Internal URL | `https://traefik.eirdom.homes` |
| External URL | Internal only |
| Container Port | 80, 443, 8080 (dashboard) |
| Config Path | `${DOCKER_DATA_PATH}/traefik` |
| Networks | `proxy` |
| Credentials | Password manager — dashboard basic auth |
| Backup | Weekly — `${DOCKER_DATA_PATH}/traefik` |
| Dependencies | None — must start first |
| Health Check | `https://traefik.eirdom.homes/ping` |

**Notes:** Traefik must always start before any other Docker service.
TLS certificates are issued via Let's Encrypt DNS challenge using
Cloudflare API credentials. Wildcard cert covers `*.eirdom.homes`.
Certificate stored in `${DOCKER_DATA_PATH}/traefik/certs/acme.json`
(permissions must be 600).

---

### Cloudflared — Cloudflare Tunnel

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `cloudflared` |
| Image | `cloudflare/cloudflared:latest` |
| VLAN | 50 |
| Internal URL | N/A |
| External URL | N/A — outbound connector only |
| Container Port | None |
| Networks | `proxy` |
| Credentials | Password manager — tunnel token in `.env` |
| Backup | Token stored in password manager |
| Dependencies | Traefik |
| Health Check | Zero Trust dashboard → Tunnels → (tunnel status) |

**Notes:** Initiates outbound-only connection to Cloudflare edge.
Zero inbound ports. Configured entirely through Cloudflare Zero Trust
dashboard. Tunnel ingress rules defined in Zero Trust → Networks →
Tunnels → (tunnel) → Public Hostnames.

**Active Tunnel Ingress Rules:**

| Subdomain | Domain | Backend | Public |
|-----------|--------|---------|--------|
| `@` | `eirdom.homes` | `http://traefik:80` | Yes |
| `www` | `eirdom.homes` | `http://traefik:80` | Yes |
| `jellyfin` | `eirdom.homes` | `http://traefik:80` | Yes |
| `jellyseerr` | `eirdom.homes` | `http://traefik:80` | Yes |

---

### WordPress — Family Website

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `wordpress` |
| Image | `wordpress:latest` |
| VLAN | 50 |
| Internal URL | `https://eirdom.homes` |
| External URL | `https://eirdom.homes` |
| Container Port | 80 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/wordpress/html` |
| Networks | `proxy`, `wordpress-internal` |
| Credentials | Password manager |
| Backup | Daily — DB dump + files |
| Dependencies | MariaDB, Traefik |
| Health Check | `https://eirdom.homes` → HTTP 200 |

---

### MariaDB — WordPress Database

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `mariadb` |
| Image | `mariadb:latest` |
| VLAN | 50 |
| Internal URL | Internal only — no web UI |
| External URL | Internal only |
| Container Port | 3306 (internal network only) |
| Config Path | `${DOCKER_DATA_PATH}/wordpress/db` |
| Networks | `wordpress-internal` |
| Credentials | Password manager — root + wordpress user |
| Backup | Daily — `mysqldump` via backup.sh |
| Dependencies | None |
| Health Check | `docker exec mariadb mysqladmin ping` |

**Notes:** MariaDB is on the isolated `wordpress-internal` network
and is not reachable from the `proxy` network or any other container
outside the WordPress stack.

---

### Jellyfin — Media Server

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `jellyfin` |
| Image | `lscr.io/linuxserver/jellyfin:latest` |
| VLAN | 50 |
| Internal URL | `https://jellyfin.eirdom.homes` |
| External URL | `https://jellyfin.eirdom.homes` |
| Container Port | 8096 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/jellyfin` |
| Media Path | `${MEDIA_PATH}` |
| Networks | `proxy`, `jellyfin-internal` |
| Auth | AD LDAP via LDAP Authentication plugin → EIRDOM-DC-01 |
| Credentials | AD domain credentials (Jellyfin-Users group) |
| Backup | Daily — config only (media is not backed up) |
| Dependencies | Traefik |
| Health Check | `https://jellyfin.eirdom.homes/health` |

**Notes:** Transcoding is **disabled** — direct play only. The server
(Intel Xeon X3430) has no Quick Sync or VAAPI support. Clients must
be capable of native HEVC/H.265 playback for 4K content. Recommended
clients: Apple TV 4K, Fire TV Stick 4K, Shield TV. Authentication is
handled directly via AD LDAP — no Authentik ForwardAuth. Users must
be members of the `Jellyfin-Users` AD group to log in.

---

### Jellyseerr — Media Request Management

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `jellyseerr` |
| Image | `fallenbagel/jellyseerr:latest` |
| VLAN | 50 |
| Internal URL | `https://requests.eirdom.homes` |
| External URL | `https://requests.eirdom.homes` |
| Container Port | 5055 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/jellyseerr` |
| Networks | `proxy`, `jellyfin-internal` |
| Auth | Jellyfin account SSO (AD users via Jellyfin LDAP) |
| Credentials | AD domain credentials |
| Backup | Daily — config |
| Dependencies | Traefik, Jellyfin, Radarr, Radarr-4K, Sonarr, Sonarr-4K |
| Health Check | `https://requests.eirdom.homes` → HTTP 200 |

---

### Radarr — HD Movie Management

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `radarr` |
| Image | `lscr.io/linuxserver/radarr:latest` |
| VLAN | 50 |
| Internal URL | `https://radarr.eirdom.homes` |
| External URL | Internal only |
| Container Port | 7878 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/radarr` |
| Media Path | `${MEDIA_PATH}/radarr` |
| Networks | `proxy`, `arr-internal` |
| Credentials | Password manager — API key in `arr-stack/.env` |
| Backup | Daily — config |
| Dependencies | Traefik, Prowlarr, Gluetun |
| Health Check | `https://radarr.eirdom.homes` → HTTP 200 |

**Notes:** HD movies (1080p and below). Download client host is
`gluetun:8080` — not `qbittorrent`. Quality profiles managed by
Recyclarr (HD-1080p profile).

---

### Radarr 4K — 4K Movie Management

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `radarr-4k` |
| Image | `lscr.io/linuxserver/radarr:latest` |
| VLAN | 50 |
| Internal URL | `https://radarr4k.eirdom.homes` |
| External URL | Internal only |
| Container Port | 7878 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/radarr-4k` |
| Media Path | `${MEDIA_PATH}/radarr-4k` |
| Networks | `proxy`, `arr-internal` |
| Credentials | Password manager — API key in `arr-stack/.env` |
| Backup | Daily — config |
| Dependencies | Traefik, Prowlarr, Gluetun |
| Health Check | `https://radarr4k.eirdom.homes` → HTTP 200 |

**Notes:** 4K/UHD movies (2160p). Separate instance from Radarr HD
to allow independent quality profiles. Prefers Dolby Vision + TrueHD
Atmos. Quality profiles managed by Recyclarr (4K-2160p profile).

---

### Sonarr — HD TV Show Management

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `sonarr` |
| Image | `lscr.io/linuxserver/sonarr:latest` |
| VLAN | 50 |
| Internal URL | `https://sonarr.eirdom.homes` |
| External URL | Internal only |
| Container Port | 8989 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/sonarr` |
| Media Path | `${MEDIA_PATH}/sonarr` |
| Networks | `proxy`, `arr-internal` |
| Credentials | Password manager — API key in `arr-stack/.env` |
| Backup | Daily — config |
| Dependencies | Traefik, Prowlarr, Gluetun |
| Health Check | `https://sonarr.eirdom.homes` → HTTP 200 |

**Notes:** HD TV shows (1080p and below). Download client host is
`gluetun:8080`. Quality profiles managed by Recyclarr (HD-1080p).

---

### Sonarr 4K — 4K TV Show Management

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `sonarr-4k` |
| Image | `lscr.io/linuxserver/sonarr:latest` |
| VLAN | 50 |
| Internal URL | `https://sonarr4k.eirdom.homes` |
| External URL | Internal only |
| Container Port | 8989 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/sonarr-4k` |
| Media Path | `${MEDIA_PATH}/sonarr-4k` |
| Networks | `proxy`, `arr-internal` |
| Credentials | Password manager — API key in `arr-stack/.env` |
| Backup | Daily — config |
| Dependencies | Traefik, Prowlarr, Gluetun |
| Health Check | `https://sonarr4k.eirdom.homes` → HTTP 200 |

**Notes:** 4K/UHD TV shows (2160p). Separate instance from Sonarr HD.
Quality profiles managed by Recyclarr (4K-2160p profile).

---

### Lidarr — Music Management

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `lidarr` |
| Image | `lscr.io/linuxserver/lidarr:latest` |
| VLAN | 50 |
| Internal URL | `https://lidarr.eirdom.homes` |
| External URL | Internal only |
| Container Port | 8686 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/lidarr` |
| Media Path | `${MEDIA_PATH}/lidarr` |
| Networks | `proxy`, `arr-internal` |
| Credentials | Password manager — API key in `arr-stack/.env` |
| Backup | Daily — config |
| Dependencies | Traefik, Prowlarr, Gluetun |
| Health Check | `https://lidarr.eirdom.homes` → HTTP 200 |

---

### Prowlarr — Indexer Management

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `prowlarr` |
| Image | `lscr.io/linuxserver/prowlarr:latest` |
| VLAN | 50 |
| Internal URL | `https://prowlarr.eirdom.homes` |
| External URL | Internal only |
| Container Port | 9696 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/prowlarr` |
| Networks | `proxy`, `arr-internal` |
| Credentials | Password manager — API key in `arr-stack/.env` |
| Backup | Daily — config |
| Dependencies | Traefik |
| Health Check | `https://prowlarr.eirdom.homes` → HTTP 200 |

**Notes:** Prowlarr is the central indexer manager. It syncs indexer
configurations to Radarr, Sonarr, and Lidarr automatically. Configure
indexers in Prowlarr only — do not add indexers directly in individual
ARR apps.

---

### Gluetun — WireGuard VPN Client

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `gluetun` |
| Image | `qmcgaw/gluetun:latest` |
| VLAN | 50 |
| Internal URL | N/A — network gateway only |
| External URL | N/A |
| Container Port | 8080 (qBittorrent Web UI proxied through Gluetun) |
| Config Path | `${DOCKER_DATA_PATH}/gluetun` |
| Networks | `proxy`, `arr-internal` |
| Credentials | TorGuard WireGuard keys in `arr-stack/.env` |
| Backup | Credentials stored in password manager |
| Dependencies | Traefik |
| Health Check | `docker exec gluetun /gluetun-entrypoint healthcheck` |

**Notes:** TorGuard WireGuard VPN. qBittorrent shares Gluetun's
network namespace via `network_mode: service:gluetun`. Kill switch
enforced — if VPN drops, qBittorrent loses all network access. Traefik
labels for the qBittorrent Web UI live on this container, not on
qBittorrent. ARR apps reach qBittorrent at `http://gluetun:8080`.

---

### qBittorrent — Torrent Download Client

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `qbittorrent` |
| Image | `lscr.io/linuxserver/qbittorrent:latest` |
| VLAN | 50 (via Gluetun — all traffic through VPN) |
| Internal URL | `https://qbit.eirdom.homes` |
| External URL | Internal only |
| Container Port | 8080 (via Gluetun network namespace) |
| Config Path | `${DOCKER_DATA_PATH}/qbittorrent` |
| Networks | None — `network_mode: service:gluetun` |
| Credentials | Password manager |
| Backup | Daily — config |
| Dependencies | Gluetun (must be healthy before qBittorrent starts) |
| Health Check | `https://qbit.eirdom.homes` → HTTP 200 |

**Notes:** Has no independent network interface. All traffic routed
through Gluetun VPN tunnel. Seeding ratio: 1.0 → auto-remove torrent
and files. Category-based download folders per ARR app. Hardlinks
enabled — deleted downloads do not affect media library.

---

### Recyclarr — TRaSH Guides Quality Profile Sync

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `recyclarr` |
| Image | `ghcr.io/recyclarr/recyclarr:latest` |
| VLAN | 50 |
| Internal URL | N/A — no web UI |
| External URL | N/A |
| Config Path | `${DOCKER_DATA_PATH}/recyclarr` |
| Networks | `arr-internal` |
| Credentials | API keys in `${DOCKER_DATA_PATH}/recyclarr/secrets.yml` |
| Backup | Daily — config |
| Dependencies | Radarr, Radarr-4K, Sonarr, Sonarr-4K |

**Notes:** Syncs TRaSH Guides quality profiles and custom formats to
all four ARR instances. Manages HD-1080p and 4K-2160p profiles.
Runs on a schedule — no manual intervention needed after initial setup.

---

### FlareSolverr — Cloudflare Challenge Bypass

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `flaresolverr` |
| Image | `ghcr.io/flaresolverr/flaresolverr:latest` |
| VLAN | 50 |
| Internal URL | `http://flaresolverr:8191` (internal only) |
| External URL | Not exposed |
| Container Port | 8191 (arr-internal network only) |
| Networks | `arr-internal` |
| Credentials | None |
| Backup | None — stateless |
| Dependencies | None |

**Notes:** Internal service only — no Traefik route. Prowlarr connects
to it directly at `http://flaresolverr:8191` to bypass Cloudflare
protection on torrent indexer sites.

---

### Jellystat — Jellyfin Statistics

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `jellystat` |
| Image | `cyfershepard/jellystat:latest` |
| VLAN | 50 |
| Internal URL | `https://jellystat.eirdom.homes` |
| External URL | Internal only |
| Container Port | 3000 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/jellystat` |
| Networks | `proxy`, `jellyfin-internal` |
| Auth | Authentik ForwardAuth (chain-admin — VLAN 10 only) |
| Credentials | Password manager — Jellyfin API key in `jellyfin/.env` |
| Backup | Daily — DB dump (PostgreSQL) + backup-data dir |
| Dependencies | Traefik, Jellyfin, jellystat-db |
| Health Check | `https://jellystat.eirdom.homes` → HTTP 200 |

---

### Authentik — Identity Provider / SSO

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Containers | `authentik-server`, `authentik-worker` |
| Image | `ghcr.io/goauthentik/server:2026.2` |
| VLAN | 50 |
| Internal URL | `https://auth.eirdom.homes` |
| External URL | Internal only |
| Container Port | 9000 |
| Config Path | `${DOCKER_DATA_PATH}/authentik` |
| Networks | `proxy`, `authentik-internal` |
| Credentials | Password manager — admin account |
| Backup | Daily — PostgreSQL dump + media dir |
| Dependencies | Traefik, authentik-postgres |
| Health Check | `https://auth.eirdom.homes` → HTTP 200 |

**Notes:** SSO provider for all internal services via Traefik
ForwardAuth. Syncs users and groups from AD via LDAP (EIRDOM-DC-01).
No Redis required — fully PostgreSQL-based since v2025.10.
Services using Authentik SSO: Traefik dashboard, Prowlarr, Radarr,
Sonarr, Lidarr, Bazarr, qBittorrent, NetBox*, Jellystat.
(*NetBox uses AD LDAP directly, not Authentik ForwardAuth.)

---

### NetBox — Network Documentation & IPAM

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Containers | `netbox`, `netbox-worker`, `netbox-housekeeping` |
| Image | `eirdom/netbox:v4.5` (custom — netbox_unifi_sync plugin) |
| VLAN | 50 |
| Internal URL | `https://netbox.eirdom.homes` |
| External URL | Internal only |
| Container Port | 8080 |
| Config Path | `${DOCKER_DATA_PATH}/netbox` |
| Networks | `proxy`, `netbox-internal` |
| Auth | AD LDAP direct (not Authentik ForwardAuth) |
| Credentials | AD domain credentials (NetBox-Users group) |
| Backup | Daily — PostgreSQL dump + media/reports/scripts |
| Dependencies | Traefik, netbox-postgres, netbox-redis |
| Health Check | `https://netbox.eirdom.homes/api/` → HTTP 200 |

**Notes:** Custom Docker image with `netbox_unifi_sync` plugin. Syncs
UniFi devices, VLANs, WLANs, and DHCP scopes from UDM-Pro-Max into
NetBox automatically. Source of truth for all network device inventory
and IP address management.

---

### Bazarr — Subtitle Management

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `bazarr` |
| Image | `lscr.io/linuxserver/bazarr:latest` |
| VLAN | 50 |
| Internal URL | `https://bazarr.eirdom.homes` |
| External URL | Internal only |
| Container Port | 6767 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/bazarr` |
| Media Path | `${MEDIA_PATH}` |
| Networks | `proxy`, `arr-internal` |
| Credentials | Password manager — API key in `arr-stack/.env` |
| Backup | Daily — config |
| Dependencies | Traefik, Radarr, Radarr-4K, Sonarr, Sonarr-4K |
| Health Check | `https://bazarr.eirdom.homes` → HTTP 200 |

---

### Uptime Kuma — Service Monitoring

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `uptime-kuma` |
| Image | `louislam/uptime-kuma:latest` |
| VLAN | 50 |
| Internal URL | `https://status.eirdom.homes` |
| External URL | Internal only |
| Container Port | 3001 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/uptime-kuma` |
| Networks | `proxy` |
| Auth | Authentik ForwardAuth (chain-standard) |
| Credentials | Password manager — internal Uptime Kuma login |
| Backup | Daily — config (SQLite DB included in data dir) |
| Dependencies | Traefik |
| Health Check | `https://status.eirdom.homes` → HTTP 200 |

**Notes:** Monitors all Eirdom services via HTTP, TCP, and Docker
socket. Has access to `/var/run/docker.sock` (read-only) for container
health monitoring. Configure notification channels (email, push) after
deployment so alerts actually reach you when something goes down.

---

### Stirling PDF — PDF Tools

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `stirling-pdf` |
| Image | `frooodle/s-pdf:latest` |
| VLAN | 50 |
| Internal URL | `https://pdf.eirdom.homes` |
| External URL | Internal only |
| Container Port | 8080 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/stirling-pdf/configs` |
| Networks | `proxy` |
| Auth | Authentik ForwardAuth (chain-standard) |
| Credentials | None — Authentik handles auth |
| Backup | Weekly — config only (stateless, no user data) |
| Dependencies | Traefik |
| Health Check | `https://pdf.eirdom.homes` → HTTP 200 |

**Notes:** Fully stateless — no user data is stored between sessions.
All PDF processing happens in-memory. Replaces every online PDF tool
(SmallPDF, iLovePDF etc.) — nothing leaves the network. Features:
merge, split, compress, rotate, OCR, convert to/from Word/Excel,
watermark, redact, and more. Max file size set to 100MB for large
architectural plans and documents.

---

### Paperless-ngx — Document Management

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `paperless` |
| Image | `ghcr.io/paperless-ngx/paperless-ngx:latest` |
| VLAN | 50 |
| Internal URL | `https://paperless.eirdom.homes` |
| External URL | Internal only |
| Container Port | 8000 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/paperless/data` |
| Consume Path | `${MEDIA_PATH}/paperless/consume` |
| Export Path | `${MEDIA_PATH}/paperless/export` |
| Networks | `proxy`, `paperless-internal` |
| Auth | Authentik ForwardAuth + header passthrough |
| Credentials | AD domain credentials via Authentik |
| Backup | Daily — DB dump + data dir + media dir |
| Dependencies | Traefik, Authentik, paperless-db, paperless-redis |
| Health Check | `https://paperless.eirdom.homes` → HTTP 200 |

**Notes:** Documents dropped into `${MEDIA_PATH}/paperless/consume/`
are automatically OCR'd, tagged, and indexed. Authentik passes the
authenticated username via `X-Authentik-Username` header so Paperless
auto-logs in the correct user — no separate Paperless login needed.
Delete `PAPERLESS_ADMIN_USER` and `PAPERLESS_ADMIN_PASSWORD` from
`.env` after first login.

---

### Immich — Photo & Video Backup

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Containers | `immich-server`, `immich-machine-learning` |
| Image | `ghcr.io/immich-app/immich-server:release` |
| VLAN | 50 |
| Internal URL | `https://photos.eirdom.homes` |
| External URL | `https://photos.eirdom.homes` |
| Container Port | 2283 (internal, via Traefik) |
| Library Path | `${MEDIA_PATH}/immich/library` |
| Config Path | `${DOCKER_DATA_PATH}/immich` |
| Networks | `proxy`, `immich-internal` |
| Auth | Immich native (email + password) |
| Credentials | Password manager — admin account |
| Backup | Daily — DB dump (media library not backed up) |
| Dependencies | Traefik, immich-db, immich-redis |
| Health Check | `https://photos.eirdom.homes` → HTTP 200 |

**Notes:** Uses `chain-public` — Immich mobile apps cannot use
Authentik ForwardAuth, so Immich handles its own authentication.
The first account registered becomes the admin — create it before
sharing the URL with family. ML container handles face recognition
and CLIP semantic search (CPU-only on this hardware — slower than
GPU but fully functional). Media library is not backed up by
`backup.sh` — photos exist on family devices and can be re-uploaded.
The DB backup preserves albums, faces, tags, and all metadata.

---

### Ntfy — Push Notifications

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `ntfy` |
| Image | `binwiederhier/ntfy:latest` |
| VLAN | 50 |
| Internal URL | `https://ntfy.eirdom.homes` |
| External URL | Internal only |
| Container Port | 80 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/ntfy` |
| Networks | `proxy` |
| Auth | Authentik ForwardAuth (web UI) + token auth (services/apps) |
| Credentials | Password manager — admin user + service tokens |
| Backup | Daily — SQLite cache and auth DB |
| Dependencies | Traefik |
| Health Check | `https://ntfy.eirdom.homes` → HTTP 200 |

**Notes:** Provides push notifications to phones and browsers for
Uptime Kuma alerts, Wazuh critical events, and `backup.sh` completion.
Authentication is two-layer — Authentik ForwardAuth protects the web
UI, and ntfy's own token auth is used by services and mobile apps
(mobile apps use tokens directly, not ForwardAuth). Create an admin
user and per-service access tokens after deployment.

---

### Homebox — Home Inventory

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `homebox` |
| Image | `ghcr.io/sysadminsmedia/homebox:latest` |
| VLAN | 50 |
| Internal URL | `https://homebox.eirdom.homes` |
| External URL | Internal only |
| Container Port | 7745 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/homebox` |
| Networks | `proxy` |
| Auth | Authentik ForwardAuth (chain-standard) |
| Credentials | Password manager — admin account |
| Backup | Daily — SQLite database included in config dir |
| Dependencies | Traefik |
| Health Check | `https://homebox.eirdom.homes` → HTTP 200 |

**Notes:** Tracks all home assets — appliances, tools, electronics —
with purchase dates, serial numbers, warranty periods, and locations.
Pairs with Paperless-ngx for warranty document linking. Especially
valuable from day one in a new home build before items accumulate
without records.

---

### Grocy — Home Inventory & Grocery Management

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `grocy` |
| Image | `lscr.io/linuxserver/grocy:latest` |
| VLAN | 50 |
| Internal URL | `https://grocy.eirdom.homes` |
| External URL | Internal only |
| Container Port | 80 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/grocy` |
| Networks | `proxy` |
| Auth | Authentik ForwardAuth (chain-standard) |
| Credentials | Password manager — admin account |
| Backup | Daily — SQLite database included in config dir |
| Dependencies | Traefik |
| Health Check | `https://grocy.eirdom.homes` → HTTP 200 |

**Notes:** Tracks pantry, fridge, and freezer inventory with barcode
scanning and expiry date tracking. Generates shopping lists from stock
levels. Supports household task management (chores, recurring home
maintenance). Default credentials are `admin`/`admin` — change
immediately on first login.

---

### Mealie — Recipe Management & Meal Planning

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `mealie` |
| Image | `ghcr.io/mealie-recipes/mealie:latest` |
| VLAN | 50 |
| Internal URL | `https://mealie.eirdom.homes` |
| External URL | Internal only |
| Container Port | 9000 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/mealie/data` |
| Networks | `proxy`, `mealie-internal` |
| Auth | Authentik ForwardAuth (chain-standard) |
| Credentials | Password manager — admin account |
| Backup | Daily — DB dump + data dir |
| Dependencies | Traefik, mealie-db |
| Health Check | `https://mealie.eirdom.homes` → HTTP 200 |

**Notes:** Central family recipe library with automatic recipe import
from any URL. Weekly meal planning with calendar view and shopping list
generation. Integrates with Grocy for pantry-aware lists. `ALLOW_SIGNUP`
is set to `false` — the admin invites family members via the admin UI.

---

### Actual Budget — Personal Finance

| Field | Value |
|-------|-------|
| Status | 🟢 Production |
| Container | `actual` |
| Image | `actualbudget/actual-server:latest` |
| VLAN | 50 |
| Internal URL | `https://actual.eirdom.homes` |
| External URL | Internal only |
| Container Port | 5006 (internal, via Traefik) |
| Config Path | `${DOCKER_DATA_PATH}/actual` |
| Networks | `proxy` |
| Auth | Actual Budget native (server password) |
| Credentials | Password manager — server password |
| Backup | Daily — SQLite database included in data dir |
| Dependencies | Traefik |
| Health Check | `https://actual.eirdom.homes` → HTTP 200 |

**Notes:** Uses `chain-public` — financial data should require explicit
login rather than Authentik auto-login via header passthrough. On first
access a server password is set; this is required each time a new
device connects. Supports iOS, Android, and desktop clients. All data
stays on the server — zero cloud sync.

---

| Network | Driver | Scope | Connected Services |
|---------|--------|-------|-------------------|
| `proxy` | Bridge | Internal | Traefik, Cloudflared, WordPress, Authentik, NetBox, Jellyfin, Jellyseerr, Jellystat, Gluetun, all ARR apps, Uptime Kuma, Stirling PDF, Paperless, Immich, Ntfy, Homebox, Grocy, Mealie, Actual |
| `wordpress-internal` | Bridge | Internal | WordPress, MariaDB |
| `authentik-internal` | Bridge | Internal | Authentik server/worker, authentik-postgres |
| `netbox-internal` | Bridge | Internal | NetBox, netbox-postgres, netbox-redis |
| `arr-internal` | Bridge | Internal | Gluetun, qBittorrent, Radarr, Radarr-4K, Sonarr, Sonarr-4K, Lidarr, Prowlarr, Bazarr, FlareSolverr, Recyclarr |
| `jellyfin-internal` | Bridge | Internal | Jellyfin, Jellyseerr, Jellystat, jellystat-db |
| `paperless-internal` | Bridge | Internal | Paperless, paperless-db, paperless-redis |
| `immich-internal` | Bridge | Internal | immich-server, immich-machine-learning, immich-db, immich-redis |
| `mealie-internal` | Bridge | Internal | Mealie, mealie-db |
| `monitoring` | Bridge | Internal | Reserved for future monitoring containers |

---

## Service Dependency Chain

Start services in this order. Stopping should be done in reverse.

```
Traefik                               ← Start first, always
    └── Authentik                     ← SSO for all internal services
    └── Cloudflared                   ← Depends on Traefik
    └── MariaDB
        └── WordPress                 ← Depends on MariaDB + Traefik
    └── Gluetun (VPN)                 ← Must be healthy before qBittorrent
        └── qBittorrent               ← Shares Gluetun network namespace
    └── Prowlarr
        └── Radarr                    ← Depends on Prowlarr + Gluetun
        └── Radarr-4K                 ← Depends on Prowlarr + Gluetun
        └── Sonarr                    ← Depends on Prowlarr + Gluetun
        └── Sonarr-4K                 ← Depends on Prowlarr + Gluetun
        └── Lidarr                    ← Depends on Prowlarr + Gluetun
            └── Bazarr                ← Depends on Radarr + Sonarr (all 4)
            └── Recyclarr             ← Depends on Radarr + Sonarr (all 4)
    └── Jellyfin
        └── Jellyseerr                ← Depends on Jellyfin
        └── jellystat-db
            └── Jellystat             ← Depends on jellystat-db + Jellyfin
    └── authentik-postgres
        └── Authentik                 ← Depends on postgres
    └── netbox-postgres + netbox-redis
        └── NetBox                    ← Depends on postgres + redis
```

---

## Internal DNS Records (EIRDOM-DC-01)

All records are A records pointing to `10.1.50.10` (Traefik) except
where noted. Configured in DNS Manager on EIRDOM-DC-01 under the
`eirdom.homes` forward lookup zone.

| Hostname | Record Type | Value | Service |
|----------|------------|-------|---------|
| `eirdom.homes` | A | 10.1.50.10 | WordPress |
| `www` | CNAME | `eirdom.homes` | WordPress |
| `traefik` | A | 10.1.50.10 | Traefik Dashboard |
| `auth` | A | 10.1.50.10 | Authentik |
| `netbox` | A | 10.1.50.10 | NetBox |
| `jellyfin` | A | 10.1.50.10 | Jellyfin |
| `requests` | A | 10.1.50.10 | Jellyseerr |
| `jellystat` | A | 10.1.50.10 | Jellystat |
| `radarr` | A | 10.1.50.10 | Radarr HD |
| `radarr4k` | A | 10.1.50.10 | Radarr 4K |
| `sonarr` | A | 10.1.50.10 | Sonarr HD |
| `sonarr4k` | A | 10.1.50.10 | Sonarr 4K |
| `lidarr` | A | 10.1.50.10 | Lidarr |
| `prowlarr` | A | 10.1.50.10 | Prowlarr |
| `bazarr` | A | 10.1.50.10 | Bazarr |
| `qbit` | A | 10.1.50.10 | qBittorrent |
| `status` | A | 10.1.50.10 | Uptime Kuma |
| `pdf` | A | 10.1.50.10 | Stirling PDF |
| `paperless` | A | 10.1.50.10 | Paperless-ngx |
| `photos` | A | 10.1.50.10 | Immich |
| `ntfy` | A | 10.1.50.10 | Ntfy |
| `homebox` | A | 10.1.50.10 | Homebox |
| `grocy` | A | 10.1.50.10 | Grocy |
| `mealie` | A | 10.1.50.10 | Mealie |
| `actual` | A | 10.1.50.10 | Actual Budget |
| `proxmox` | A | 10.1.10.5 | Proxmox Web UI |
| `wazuh` | A | 10.1.60.10 | Wazuh Dashboard |
| `securityonion` | A | 10.1.60.20 | Security Onion |
| `homeassistant` | A | 10.1.50.10 | Home Assistant |
| `pki` | CNAME | `EIRDOM-SUB-01.ad.eirdom.homes` | PKI CRL/AIA |

---

## Cloudflare DNS Records (Public)

All public records are proxied through Cloudflare (orange cloud).
Your home IP is never exposed in any DNS record.

| Name | Type | Value | Proxied | Service |
|------|------|-------|---------|---------|
| `eirdom.homes` | A | Cloudflare Tunnel | Yes | WordPress |
| `www` | CNAME | `eirdom.homes` | Yes | WordPress |
| `jellyfin` | CNAME | `eirdom.homes` | Yes | Jellyfin |
| `requests` | CNAME | `eirdom.homes` | Yes | Jellyseerr |
| `photos` | CNAME | `eirdom.homes` | Yes | Immich |

> All other subdomains (`radarr`, `sonarr`, `traefik`, etc.) should
> have DNS records in Cloudflare set to **DNS only** (grey cloud) if
> they exist at all. This ensures they resolve internally via
> split-horizon but return no useful result from the public internet.

---

## Backup Summary

| Service | Frequency | Method | Destination |
|---------|-----------|--------|-------------|
| WordPress files | Daily | tar.gz | NAS |
| MariaDB | Daily | mysqldump | NAS |
| Jellyfin config | Daily | tar.gz | NAS |
| Jellyseerr config | Daily | tar.gz | NAS |
| Uptime Kuma config | Daily | tar.gz | NAS |
| Stirling PDF config | Weekly | tar.gz | NAS |
| Paperless data + media | Daily | tar.gz | NAS |
| Paperless DB | Daily | pg_dump | NAS |
| Immich DB | Daily | pg_dump | NAS |
| Ntfy data | Daily | tar.gz | NAS |
| Homebox data | Daily | tar.gz | NAS |
| Grocy config | Daily | tar.gz | NAS |
| Mealie data | Daily | tar.gz | NAS |
| Mealie DB | Daily | pg_dump | NAS |
| Actual Budget data | Daily | tar.gz | NAS |
| Traefik config + certs | Weekly | tar.gz | NAS |
| Cloudflared | On token change | Password manager | Password manager |
| Compose files | On change | Git | GitHub (private) |
| EIRDOM-DC-01 | Daily | Proxmox backup | NAS |
| EIRDOM-SUB-01 | Weekly | Proxmox backup | NAS |
| EIRDOM-RCA-01 | After any change | Encrypted USB | Offsite |
| EIRDOM-WAZUH-01 | Weekly | Proxmox backup | NAS |
| EIRDOM-SONION-01 | Weekly (config only) | Proxmox backup | NAS |

---

## Maintenance Schedule

| Frequency | Task | Services |
|-----------|------|---------|
| Daily | Review Wazuh alerts | Wazuh |
| Daily | Review Security Onion SOC alerts | Security Onion |
| Weekly | Run `update.sh` — pull latest images | All Docker |
| Weekly | Check Traefik dashboard for routing errors | Traefik |
| Weekly | Check Cloudflare Tunnel health | Cloudflared |
| Weekly | Review Docker stack health (`docker compose ps`) | All Docker |
| Weekly | Review WAN deny logs | UDM-Pro-Max |
| Monthly | Verify zero open inbound ports (external portscan) | UDM-Pro-Max |
| Monthly | Check PKI certificate expiration dates | EIRDOM-SUB-01 |
| Monthly | Rotate ARR stack API keys | ARR stack |
| Monthly | Review and prune IPS suppression list | UDM-Pro-Max |
| Quarterly | Review UniFi firmware updates | All UniFi |
| Quarterly | Review and update GeoIP block list | UDM-Pro-Max |
| Quarterly | Review DoD IP allocations | UDM-Pro-Max |
| Quarterly | Test AD disaster recovery | EIRDOM-DC-01 |
| Quarterly | Review firewall rules for drift | UDM-Pro-Max |
| Annually | Power on Offline Root CA, publish new CRL, shut down | EIRDOM-RCA-01 |
| Annually | Renew `eirdom.homes` domain registration | Cloudflare |
| Annually | Full infrastructure documentation review | All |

---

## Related Documentation

- [`vlans.md`](vlans.md) — VLAN reference and subnet table
- [`wireless.md`](wireless.md) — SSID and AP configuration
- [`port-profiles.md`](port-profiles.md) — Switch port profiles
- [`lan-rules.md`](lan-rules.md) — Zone-based LAN firewall rules
- [`wan-rules.md`](wan-rules.md) — WAN firewall rules
- [`ids-ips.md`](ids-ips.md) — IDS/IPS configuration
- [`Eirdom_Infrastructure_Guide_v3.docx`](../docs/Eirdom_Infrastructure_Guide_v3.docx) — Full infrastructure guide