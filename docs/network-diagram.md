# Eirdom — Network Diagram
> Last Updated: April 2026

---

## Full Network Topology

```
┌─────────────────────────────────────────────────────────────────────┐
│                            INTERNET                                 │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Cloudflare Edge   │
                    │  TLS · WAF · DDoS   │
                    │   eirdom.homes      │
                    └──────────┬──────────┘
                               │ Cloudflare Tunnel (outbound only)
┌──────────────────────────────▼──────────────────────────────────────┐
│                        UDM-Pro-Max                                  │
│                         10.1.1.1                                    │
│          Gateway · Firewall · Controller · Protect NVR              │
│              IDS/IPS · GeoIP · Zone Firewall                        │
└───┬─────────────────────────────────────────────────────────────────┘
    │ 10G SFP+
    │
┌───▼──────────────────────────────────────────────────────────────────┐
│              USW-Pro-Max-48-PoE — CORE (10.1.1.2)                    │
│         720W PoE · 16x 2.5GbE PoE++ · Etherlighting                  │
│                        Garage                                        │
└───┬──────────────────┬──────────────────┬────────────────────────────┘
    │ 10G SFP+         │ PoE+ / PoE++     │ Cat6 Home-Runs
    │                  │                  │
    │         ┌────────┴─────────┐        │
    │         │   Access Points  │        │
    │         │                  │        │
    │         │  AP-MAIN-01      │        │
    │         │  U7 Pro          │        │
    │         │  Living Room     │        │
    │         │                  │        │
    │         │  AP-MAIN-02      │        │
    │         │  U7 Pro          │        │
    │         │  Hallway         │        │
    │         │                  │        │
    │         │  AP-WALL-01      │        │
    │         │  U7 Pro Wall     │        │
    │         │  Office          │        │
    │         │                  │        │
    │         │  AP-OUT-01       │        │
    │         │  U7 Pro Outdoor  │        │
    │         │  Garage/Exterior │        │
    │         │                  │        │
    │         │  AP-LITE-01      │        │
    │         │  U7 Lite         │        │
    │         │  Bedroom Wing    │        │
    │         │                  │        │
    │         │  AP-LITE-02      │        │
    │         │  U7 Lite         │        │
    │         │  Server Room     │        │
    │         └──────────────────┘        │
    │                                     │
    │                           ┌─────────┴──────────────────────────┐
    │                           │        House Cat6 Drops            │
    │                           │  Wall plates · Workstations · IoT  │
    │                           └────────────────────────────────────┘
    │
┌───▼──────────────────────────────────────────────────────────────────┐
│          USW-Pro-Max-48-PoE — DISTRIBUTION (10.1.1.3)                │
│         720W PoE · 16x 2.5GbE PoE++ · Etherlighting                  │
│                      Garage / Server Room                            │
└───┬──────────────┬─────────────┬───────────────┬─────────────────────┘
    │ 2.5GbE       │ 2.5GbE      │ 1GbE PoE++    │ 1G
    │              │             │               │
┌───▼────┐    ┌────▼────┐   ┌────▼──────────┐  ┌─▼──────────────────┐
│EIRDOM  │    │EIRDOM   │   │ Access Hubs   │  │ Security Onion     │
│PVE-01  │    │DOCKER-01│   │ UA-Hub-Door   │  │ Monitor NIC        │
│10.1.   │    │10.1.    │   │ Front Door    │  │ (SPAN — no IP)     │
│10.5    │    │50.10    │   │ Garage Door   │  └────────────────────┘
└───┬────┘    └────┬────┘   └───────────────┘
    │              │
    │ Proxmox VMs  │ Docker Containers
    │              │
    ▼              ▼
(see VM map)  (see Docker map)
```

---

## VLAN Segmentation

```
┌──────────────────────────────────────────────────────────────────────┐
│                     UDM-Pro-Max (10.1.1.1)                           │
│                    Zone-Based Firewall                               │
├──────────┬──────────┬──────────┬──────────┬──────────┬───────────────┤
│  VLAN 1  │ VLAN 10  │ VLAN 20  │ VLAN 30  │ VLAN 40  │   VLAN 50     │
│  MGMT    │ CORP     │  IoT     │  Guest   │ Cameras  │   Docker      │
│10.1.1.0  │10.1.10.0 │10.1.20.0 │10.1.30.0 │10.1.40.0 │ 10.1.50.0     │
│   /24    │   /24    │   /24    │   /24    │   /24    │    /24        │
├──────────┴──────────┴──────────┴──────────┴──────────┴───────────────┤
│                           VLAN 60                                    │
│                          Security                                    │
│                        10.1.60.0/24                                  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Proxmox VM Layout (EIRDOM-PVE-01 — 10.1.10.5)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EIRDOM-PVE-01 (Proxmox VE 8.x)                   │
│                    10.1.10.5 — VLAN 10                              │
│                VLAN-aware bridge: vmbr0                             │
├────────────────────┬────────────────────┬───────────────────────────┤
│      VLAN 10       │      VLAN 10       │        VLAN 10            │
│   VM 100           │   VM 101           │      VM 102               │
│   EIRDOM-DC-01     │   EIRDOM-RCA-01    │    EIRDOM-SUB-01          │
│   10.1.10.10       │   AIR-GAPPED       │    10.1.10.12             │
│   AD DS · DNS      │   Offline Root CA  │    Issuing CA             │
│   DHCP · NPS       │   (powered off)    │    PKI · CRL/AIA          │
│   Win Server 2025  │   Win Server 2025  │    Win Server 2025        │
├────────────────────┴────────────────────┴───────────────────────────┤
│      VLAN 60                              VLAN 60                   │
│   VM 120                               VM 130                       │
│   EIRDOM-WAZUH-01                      EIRDOM-SONION-01             │
│   10.1.60.10                           10.1.60.20                   │
│   Wazuh Manager                        Security Onion 3.0           │
│   Indexer · Dashboard                  Standalone                   │
│   Ubuntu 26.04 LTS                     Oracle Linux 9               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Docker Container Layout (EIRDOM-DOCKER-01 — 10.1.50.10)

```
┌──────────────────────────────────────────────────────────────────────┐
│                EIRDOM-DOCKER-01 (Ubuntu 26.04 LTS)                   │
│                    10.1.50.10 — VLAN 50                              │
│           /media/arr (sda1, 2.7TB ext4) — single filesystem         │
│           All configs: /media/arr/config/<service>/                  │
│           All media:   /media/arr/<library>/                         │
├──────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                    proxy (Docker network)                    │    │
│  │                                                              │    │
│  │  ┌────────────┐  ┌─────────────┐  ┌────────────────────┐    │    │
│  │  │  Traefik   │◄─│ Cloudflared │  │    WordPress       │    │    │
│  │  │  v3.3      │  │ (outbound)  │  │   eirdom.homes     │    │    │
│  │  │  :443/:80  │  └─────────────┘  └────────┬───────────┘    │    │
│  │  └─────┬──────┘                            │                │    │
│  │        │ Routes by Host header             │ wordpress-     │    │
│  │        │                                   │ internal       │    │
│  │        │                            ┌──────▼──────────┐     │    │
│  │        │                            │    MariaDB      │     │    │
│  │        │                            └─────────────────┘     │    │
│  └────────┼─────────────────────────────────────────────────────┘    │
│           │                                                          │
│  ┌────────▼─────────────────────────────────────────────────────┐    │
│  │               authentik-internal network                     │    │
│  │  ┌──────────────────────┐  ┌──────────────────────────────┐  │    │
│  │  │  Authentik Server    │  │    authentik-postgres        │  │    │
│  │  │  + Worker  :9000     │◄─│    PostgreSQL                │  │    │
│  │  │  auth.eirdom.homes   │  └──────────────────────────────┘  │    │
│  │  └──────────────────────┘                                    │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │               netbox-internal network                        │    │
│  │  ┌───────────────────┐  ┌────────────┐  ┌────────────────┐  │    │
│  │  │  NetBox v4.5      │  │  netbox-   │  │  netbox-redis  │  │    │
│  │  │  + Worker + HK    │◄─│  postgres  │  │  (Valkey)      │  │    │
│  │  │  netbox.eirdom.   │  └────────────┘  └────────────────┘  │    │
│  │  │  UniFi sync plugin│                                      │    │
│  │  └───────────────────┘                                      │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │               jellyfin-internal network                      │    │
│  │  ┌──────────┐  ┌───────────────┐  ┌──────────────────────┐  │    │
│  │  │ Jellyfin │  │  Jellyseerr   │  │  Jellystat           │  │    │
│  │  │  :8096   │  │  :5055        │  │  :3000               │  │    │
│  │  │  AD LDAP │  │  requests.    │  │  jellystat.          │  │    │
│  │  └──────────┘  └───────────────┘  └─────────┬────────────┘  │    │
│  │                                             │               │    │
│  │                                   ┌─────────▼────────────┐  │    │
│  │                                   │  jellystat-db        │  │    │
│  │                                   │  PostgreSQL          │  │    │
│  │                                   └──────────────────────┘  │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │               arr-internal network                           │    │
│  │                                                              │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │  Gluetun (WireGuard VPN — TorGuard)                 │    │    │
│  │  │  ┌─────────────────────────────────────────────┐    │    │    │
│  │  │  │  qBittorrent :8080                          │    │    │    │
│  │  │  │  network_mode: service:gluetun              │    │    │    │
│  │  │  │  All traffic through VPN tunnel             │    │    │    │
│  │  │  │  Kill switch enforced                       │    │    │    │
│  │  │  └─────────────────────────────────────────────┘    │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  │           ↑ Download client (http://gluetun:8080)           │    │
│  │  ┌────────┴──────────────────────────────────────────┐      │    │
│  │  │  Prowlarr :9696  ──►  FlareSolverr :8191          │      │    │
│  │  │                                                    │      │    │
│  │  │  Radarr :7878    Radarr-4K :7878                  │      │    │
│  │  │  Sonarr :8989    Sonarr-4K :8989                  │      │    │
│  │  │  Lidarr :8686    Bazarr :6767                     │      │    │
│  │  │                                                    │      │    │
│  │  │  Recyclarr (cron — no web UI)                     │      │    │
│  │  └────────────────────────────────────────────────────┘      │    │
│  └──────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Public Traffic Flow

```
User (internet)
      │
      ▼
Cloudflare Edge
  · TLS termination
  · WAF inspection
  · DDoS protection
      │
      │ HTTPS (encrypted tunnel)
      ▼
cloudflared container
  EIRDOM-DOCKER-01
  VLAN 50
      │
      ▼
Traefik
  Routes by Host header:
  ┌──────────────────────────────────────────────┐
  │ eirdom.homes          → WordPress            │
  │ www.eirdom.homes      → WordPress            │
  │ jellyfin.eirdom.homes → Jellyfin             │
  │ requests.eirdom.homes → Jellyseerr           │
  └──────────────────────────────────────────────┘
```

---

## Internal Traffic Flow

```
Internal Client (VLAN 10 / 20)
      │
      │ DNS: *.eirdom.homes → 10.1.50.10
      │ (resolved by EIRDOM-DC-01 split-horizon)
      ▼
Traefik (10.1.50.10:443)
  Routes by Host header:
  ┌──────────────────────────────────────────────────────────────┐
  │ eirdom.homes              → WordPress                        │
  │ auth.eirdom.homes         → Authentik (public access)        │
  │ jellyfin.eirdom.homes     → Jellyfin  (AD LDAP auth)         │
  │ requests.eirdom.homes     → Jellyseerr (Jellyfin auth)       │
  │ netbox.eirdom.homes       → NetBox    (AD LDAP auth)         │
  │ traefik.eirdom.homes      → Traefik Dashboard (admin only)   │
  │ radarr.eirdom.homes       → Radarr HD  (Authentik SSO)       │
  │ radarr4k.eirdom.homes     → Radarr 4K  (Authentik SSO)      │
  │ sonarr.eirdom.homes       → Sonarr HD  (Authentik SSO)       │
  │ sonarr4k.eirdom.homes     → Sonarr 4K  (Authentik SSO)      │
  │ lidarr.eirdom.homes       → Lidarr     (Authentik SSO)       │
  │ prowlarr.eirdom.homes     → Prowlarr   (Authentik SSO)       │
  │ bazarr.eirdom.homes       → Bazarr     (Authentik SSO)       │
  │ qbit.eirdom.homes         → qBittorrent (Authentik SSO)      │
  │ jellystat.eirdom.homes    → Jellystat  (admin only)          │
  │ proxmox.eirdom.homes      → Proxmox    (admin only)          │
  │ wazuh.eirdom.homes        → Wazuh      (admin only)          │
  │ securityonion.eirdom.homes→ Sec. Onion (admin only)          │
  └──────────────────────────────────────────────────────────────┘

  Middleware chains:
  · chain-public  — Jellyfin, Jellyseerr, WordPress (own auth)
  · chain-standard — all other services (Authentik SSO)
  · chain-admin  — Traefik, Jellystat, Proxmox, Wazuh, Sec. Onion
                    (Authentik SSO + VLAN 10 whitelist)
```

---

## Physical Security Layout (VLAN 40)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      VLAN 40 — Cameras                              │
│               Managed by UDM-Pro-Max (Protect + Access)             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Cameras                   Access Control         Sensors           │
│  ─────────────────         ──────────────         ───────────────   │
│  G6 180 × 4                UA-Hub-Door × 2        SuperLink GW × 2  │
│  House sides               Front Door             Interior + Garage │
│  Garage sides              Garage Door                              │
│                                                                     │
│  G6 Bullet × 1             Readers (via hub)                        │
│  Garage Interior           UA-G2/G3 Pro                             │
│                                                                     │
│  G6 Turret × 1                                                      │
│  Front Door Area                                                    │
│                                                                     │
│  AI Theta × 1                                                       │
│  Interior (indoor)                                                  │
│                                                                     │
│  G4 Doorbell Pro PoE × 1                                            │
│  Front Door                                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                               │
                    All devices report to
                    UDM-Pro-Max Protect NVR
                    via TCP 7443 / 7444
```

---

## Wireless Coverage (SSIDs)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Wireless Networks                            │
├──────────────────┬──────────────────┬───────────────────────────────┤
│    Eirdom        │   Eirdom-IoT     │       Eirdom-Guest            │
│    VLAN 10       │   VLAN 20        │       VLAN 30                 │
│    WPA3-Ent      │   WPA3-Personal  │       WPA3-Personal           │
│    802.1X/NPS    │   2.4GHz only    │       2.4 + 5GHz              │
│    2.4+5+6GHz    │   Client isolate │       Client isolate          │
│    All APs       │   All APs        │       All APs                 │
└──────────────────┴──────────────────┴───────────────────────────────┘
```

---

## Security Monitoring Data Flow

```
All VLANs
    │
    ├── Syslog (UDP 514) ──────────────────► EIRDOM-WAZUH-01
    │   UDM-Pro-Max                          10.1.60.10
    │   IDS/IPS alerts                       Wazuh Manager
    │   Firewall deny logs                   + Indexer
    │                                        + Dashboard
    │
    ├── SPAN port (Layer 2 mirror) ─────────► EIRDOM-SONION-01
    │   Distribution switch port 43          10.1.60.20
    │   Mirrors UDM uplink                   Security Onion 3.0
    │   All VLAN traffic                     Passive capture NIC
    │   (passive, no IP)                     (no IP — monitor only)
    │
    └── Wazuh Agents ──────────────────────► EIRDOM-WAZUH-01
        Installed on:                        10.1.60.10
        · EIRDOM-DC-01
        · EIRDOM-SUB-01
        · EIRDOM-DOCKER-01
        · EIRDOM-PVE-01
```

---

## DNS Resolution Flow

```
                    ┌─────────────────────────────────┐
                    │      DNS Query: *.eirdom.homes  │
                    └──────────────┬──────────────────┘
                                   │
               ┌───────────────────┴──────────────────┐
               │                                      │
    Internal client                          External client
    (VLAN 10/20/50/60)                       (internet)
               │                                      │
               ▼                                      ▼
    EIRDOM-DC-01                            Cloudflare Public DNS
    10.1.10.10                              (1.1.1.1)
    Split-horizon zone                      eirdom.homes zone
               │                                       │
               ▼                                       ▼
    Returns 10.1.50.10                     Returns Cloudflare
    (Traefik — internal)                   proxy IP
               │                                       │
               ▼                                       ▼
    Traffic stays on LAN                   Traffic → Cloudflare
    Never leaves network                   → Tunnel → Traefik
```

---

## IP Address Summary

```
VLAN 1  — Management (10.1.1.0/24)
  10.1.1.1    UDM-Pro-Max
  10.1.1.2    USW-Pro-Max-48-PoE (Core)
  10.1.1.3    USW-Pro-Max-48-PoE (Distribution)

VLAN 10 — Corporate (10.1.10.0/24)
  10.1.10.5   EIRDOM-PVE-01     Proxmox Host
  10.1.10.10  EIRDOM-DC-01      AD DS · DNS · DHCP · NPS
  10.1.10.12  EIRDOM-SUB-01     Subordinate / Issuing CA

VLAN 20 — IoT (10.1.20.0/24)
  10.1.20.1   Gateway (UDM)
  DHCP pool   10.1.20.100 – 10.1.20.250

VLAN 30 — Guest (10.1.30.0/24)
  10.1.30.1   Gateway (UDM)
  DHCP pool   10.1.30.100 – 10.1.30.250

VLAN 40 — Cameras (10.1.40.0/24)
  10.1.40.1   Gateway (UDM — Protect NVR)
  DHCP pool   10.1.40.100 – 10.1.40.200
  All cameras · access hubs · SuperLink GWs via DHCP

VLAN 50 — Docker (10.1.50.0/24)
  10.1.50.10  EIRDOM-DOCKER-01  Docker Host (Traefik · Cloudflared · all services)

VLAN 60 — Security (10.1.60.0/24)
  10.1.60.10  EIRDOM-WAZUH-01   Wazuh Manager · Indexer · Dashboard
  10.1.60.20  EIRDOM-SONION-01  Security Onion 3.0
```

---

## Related Documentation

- [`vlans.md`](vlans.md) — Full VLAN reference and switch port assignments
- [`wireless.md`](wireless.md) — SSID and AP configuration
- [`services.md`](services.md) — Master services reference
- [`port-profiles.md`](port-profiles.md) — Switch port profiles
- [`lan-rules.md`](lan-rules.md) — Zone-based LAN firewall rules
- [`wan-rules.md`](wan-rules.md) — WAN firewall rules
- [`ids-ips.md`](ids-ips.md) — IDS/IPS configuration