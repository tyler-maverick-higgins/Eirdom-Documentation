# Eirdom — Architecture Decision Record
> Last Updated: 2026-07-01

---

## Overview

This document records every significant architecture, hardware, and
software decision made for the Eirdom infrastructure. Each entry
explains what was decided, why it was chosen over alternatives, and
any trade-offs accepted. This is the document you read six months from
now when you can't remember why something was set up a certain way.

---

## ⚠️ Verification Needed (added 2026-07-01)

Three inconsistencies were found during the 2026-07-01 update that require
verification against the actual system before they can be resolved. They are
NOT silently changed — each is marked inline at its ADR with `⚠️ VERIFY`.

1. **ADR-003 — Windows Server version.** Entry says Server **2025**; prior
   project context repeatedly says Server **2022**. Confirm the DC's actual OS.
2. **ADR-042 / ADR-026 — VPN provider.** History records ADR-042 as a
   TorGuard→AirVPN switch, but ADR-042 is absent from this file and ADR-026
   still describes TorGuard as current. Confirm whether ADR-042 exists elsewhere
   or must be written here, and whether ADR-026 should be marked superseded.
3. **ADR-008 vs ADR-028 — Docker host hardware.** ADR-008 justifies bare-metal
   by "Intel Quick Sync transcoding (Jellyfin)"; ADR-028 states the host is a
   2009 Xeon X3430 with no Quick Sync (and the E5-2630 v4 Proxmox host has no
   iGPU). One is inaccurate. Confirm the current Docker host hardware.

---

## Decision Template

Use this template when recording a new decision:

```
### ADR-XXX — [Short Title]
**Date:** YYYY-MM-DD
**Status:** Decided | Superseded by ADR-XXX | Under Review

**Context:**
What situation or requirement prompted this decision?

**Decision:**
What was decided?

**Alternatives Considered:**
- Option A — why it was not chosen
- Option B — why it was not chosen

**Consequences:**
What does this decision mean going forward? What becomes easier or
harder as a result? Any trade-offs accepted?

**Review Date:** (optional — set if the decision should be revisited)
```

---

## Decision Index

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| ADR-001 | Zero inbound port architecture via Cloudflare Tunnel | Decided | Apr 2026 |
| ADR-002 | Split-horizon DNS on single eirdom.homes domain | Decided | Apr 2026 |
| ADR-003 | Windows Active Directory for authentication | Decided ⚠️ verify OS ver | Apr 2026 |
| ADR-004 | Two-tier offline PKI hierarchy | Decided | Apr 2026 |
| ADR-005 | Zone-based firewall over legacy per-interface rules | Decided | Apr 2026 |
| ADR-006 | Traefik as single reverse proxy ingress | Decided | Apr 2026 |
| ADR-007 | All service traffic routed through Traefik only | Decided | Apr 2026 |
| ADR-008 | Docker on dedicated bare-metal host | Decided ⚠️ verify HW | Apr 2026 |
| ADR-009 | Proxmox VE for VM hypervisor | Decided | Apr 2026 |
| ADR-010 | USW-Pro-Max-48-PoE for both switches | Decided | Apr 2026 |
| ADR-011 | U7 series access points | Decided | Apr 2026 |
| ADR-012 | WPA3-Enterprise with 802.1X on trusted SSID | Decided | Apr 2026 |
| ADR-013 | Three-SSID wireless strategy | Decided | Apr 2026 |
| ADR-014 | G6 camera fleet with AI Theta interior | Decided | Apr 2026 |
| ADR-015 | All Cat6 home-runs to garage core switch | Decided | Apr 2026 |
| ADR-016 | UFW disabled in favor of Docker iptables management | Decided | Apr 2026 |
| ADR-017 | Docker daemon bound to 127.0.0.1 by default | Decided | Apr 2026 |
| ADR-018 | IPS enabled on all security-sensitive VLANs | Decided | Apr 2026 |
| ADR-019 | GeoIP blocking of 13 high-risk countries | Decided | Apr 2026 |
| ADR-020 | DoD IP ranges blocked at WAN | Decided | Apr 2026 |
| ADR-021 | DHCP relay to AD DC on VLAN 10, UDM shadow mode elsewhere | Decided | Apr 2026 |
| ADR-022 | Wazuh + Security Onion dual monitoring stack | Decided | Apr 2026 |
| ADR-023 | Private GitHub repo for infrastructure-as-code | Decided | Apr 2026 |
| ADR-024 | Per-service Docker Compose files over single monolithic file | Decided | Apr 2026 |
| ADR-025 | Root .env for shared vars, service .env for specific vars | Decided | Apr 2026 |
| ADR-026 | Gluetun WireGuard VPN — qBittorrent-only scope | Decided ⚠️ see ADR-042 | Apr 2026 |
| ADR-027 | Dual Radarr + dual Sonarr instances for HD and 4K | Decided | Apr 2026 |
| ADR-028 | Jellyfin direct play only — no transcoding | Decided ⚠️ verify HW | Apr 2026 |
| ADR-029 | Authentik as SSO provider — AD LDAP source | Decided (updated 2026-07-01) | Apr 2026 |
| ADR-030 | NetBox with netbox_unifi_sync plugin — custom Docker image | Decided | Apr 2026 |
| ADR-031 | Single /media/arr filesystem for all media and configs | Superseded by ADR-043 | Apr 2026 |
| ADR-032 | Uptime Kuma for service monitoring | Decided | Apr 2026 |
| ADR-033 | Stirling PDF for local PDF processing | Decided | Apr 2026 |
| ADR-034 | Paperless-ngx for document management | Decided | Apr 2026 |
| ADR-035 | Immich for family photo and video backup | Decided (auth: see ADR-046) | Apr 2026 |
| ADR-036 | Ntfy for self-hosted push notifications | Decided | Apr 2026 |
| ADR-037 | Homebox for home asset inventory | Decided | Apr 2026 |
| ADR-038 | Grocy for pantry and grocery management | Decided | Apr 2026 |
| ADR-039 | Mealie for recipe management and meal planning | Decided | Apr 2026 |
| ADR-040 | Actual Budget for personal finance tracking | Decided | Apr 2026 |
| ADR-041 | Home Assistant OS on Proxmox VM for smart home | Decided | Apr 2026 |
| ADR-042 | VPN provider change (TorGuard → AirVPN) | ⚠️ MISSING from file — verify | — |
| ADR-043 | Three-tier drive layout for the media stack | Decided (supersedes ADR-031) | Jun 2026 |
| ADR-044 | Application authentication integration model (OIDC / ForwardAuth / self-auth) | Decided | Jul 2026 |
| ADR-045 | Group-based application authorization (deny-by-default via AD group bindings) | Decided | Jul 2026 |
| ADR-046 | Immich authentication via native OIDC (delegated to Authentik) | Decided (extends ADR-035) | Jul 2026 |
| ADR-047 | chain-app middleware for self-authenticating, bursty apps; remove sslRedirect | Decided | Jul 2026 |
| ADR-048 | AD LDAP source scope and filtering | Decided (refines ADR-029) | Jul 2026 |

---

## Decisions

---

### ADR-001 — Zero Inbound Port Architecture via Cloudflare Tunnel
**Date:** April 2026
**Status:** Decided

**Context:**
The family website and select media services need to be accessible from
the internet. Traditional approaches require opening inbound ports on
the firewall and creating port forwards, exposing the home IP and
creating direct attack surface.

**Decision:**
All public internet access is provided exclusively through a Cloudflare
Tunnel. The cloudflared container initiates an outbound-only encrypted
connection to Cloudflare's edge. No inbound ports are opened on the
UDM-Pro-Max. The home IP is never exposed in any public DNS record.

**Alternatives Considered:**
- Traditional port forwarding (80/443) — rejected due to direct
  exposure of home IP, need to manage TLS at the edge, and attack
  surface on the WAN interface
- VPN with split tunneling — rejected because it requires client
  configuration on every external device and doesn't suit casual
  family access

**Consequences:**
An external portscan of the home IP returns zero open ports. Even if
the IP is discovered, there is nothing to attack. All WAF, DDoS
protection, and TLS termination is handled by Cloudflare before
traffic reaches the network. Trade-off: dependency on Cloudflare
availability for external access to public services.

---

### ADR-002 — Split-Horizon DNS on Single eirdom.homes Domain
**Date:** April 2026
**Status:** Decided

**Context:**
Services need to be accessible both internally and externally using
the same URLs. Two approaches were considered: separate internal and
external subdomains, or split-horizon DNS on a single subdomain.

**Decision:**
All services use a single `*.eirdom.homes` subdomain pattern with
split-horizon DNS. Internal clients resolve `*.eirdom.homes` to
`10.1.50.10` (Traefik) via the AD DNS split-horizon zone on
EIRDOM-DC-01. External clients resolve to Cloudflare proxy IPs.
The `.local` mDNS-reserved suffix was explicitly rejected.

**Alternatives Considered:**
- Separate `*.int.eirdom.homes` for internal — rejected because it
  requires two URLs per service, breaks mobile apps when switching
  networks, and doubles DNS record maintenance
- Using `.local` suffix — rejected because it is reserved for mDNS
  (Bonjour/Avahi) and causes DNS conflicts on some devices

**Consequences:**
One URL per service works everywhere. Internal traffic never leaves
the network. Mobile apps work seamlessly on and off the home network.
Trade-off: every internal service needs a Cloudflare DNS record (set
to DNS only, not proxied) to maintain the pattern consistently.

---

### ADR-003 — Windows Active Directory for Authentication
**Date:** April 2026
**Status:** Decided

> ⚠️ VERIFY: OS version below reads "Server 2025"; prior project context
> repeatedly says "Server 2022." Confirm the DC's actual OS and correct here.

**Context:**
The network needs centralized authentication, DNS, DHCP, Group Policy,
and RADIUS for 802.1X wireless authentication. The household includes
multiple Windows workstations.

**Decision:**
Deploy a Windows Server 2025 Active Directory domain (`ad.eirdom.homes`)
on EIRDOM-DC-01. AD provides DNS, DHCP (for VLAN 10), NPS for 802.1X
RADIUS, GPO for workstation management, and certificate auto-enrollment
via the PKI.

**Alternatives Considered:**
- FreeIPA / Samba AD — rejected due to complexity of integrating with
  Windows workstations and NPS for 802.1X
- No centralized auth — rejected because per-device management does
  not scale and WPA3-Enterprise 802.1X requires a RADIUS server

**Consequences:**
Enterprise-grade authentication and policy management for a home
network. Windows workstations join the domain and receive GPOs
automatically. Trade-off: Windows Server licensing cost and VM
overhead for a home environment.

---

### ADR-004 — Two-Tier Offline PKI Hierarchy
**Date:** April 2026
**Status:** Decided

**Context:**
Internal services need TLS certificates trusted by all domain devices.
The PKI needs to be secure enough that compromise of the issuing CA
does not compromise the root of trust.

**Decision:**
Deploy a two-tier PKI: an offline Root CA (EIRDOM-RCA-01) that is
permanently air-gapped after issuing the subordinate certificate, and
an online Issuing CA (EIRDOM-SUB-01) for day-to-day certificate
issuance. Root CA uses RSA 4096-bit / SHA-384. Issuing CA uses RSA
4096-bit / SHA-256. Root CA validity 20 years, issued certs 10 years.

**Alternatives Considered:**
- Single-tier CA — rejected because compromise of the CA private key
  would require rebuilding the entire PKI and re-trusting all devices
- Let's Encrypt only — rejected because internal services on
  non-public subdomains cannot use Let's Encrypt without DNS challenge
  complexity, and the PKI also serves 802.1X machine certificates

**Consequences:**
The root CA private key is protected by being completely offline.
Even if EIRDOM-SUB-01 is compromised, the root can be used to issue
a new subordinate CA. Trade-off: annual Root CA power-on procedure to
publish new CRL, and encrypted offline backups must be maintained.

---

### ADR-005 — Zone-Based Firewall Over Legacy Per-Interface Rules
**Date:** April 2026
**Status:** Decided

**Context:**
UniFi Network 8.x introduced a zone-based firewall as an alternative
to the legacy per-VLAN interface rule approach. A decision was needed
on which model to use.

**Decision:**
Use the UniFi zone-based firewall exclusively. Networks are grouped
into named zones (CORPORATE, IOT, GUEST, CAMERAS, DOCKER, SECURITY,
MANAGEMENT) and rules are written zone-to-zone.

**Alternatives Considered:**
- Legacy per-interface rules — rejected because zone-based is cleaner,
  more readable, and is the direction UniFi is investing in for future
  development. Legacy rules would need to be migrated eventually anyway.

**Consequences:**
Rules are significantly easier to read, audit, and maintain. Zone
groupings make the security intent immediately clear. Trade-off: fewer
community examples available for zone-based firewall vs the legacy
model which has years of documented configurations online.

**Note (2026-07-01):** Block rules must match on New/Invalid connection
state, not All — matching All can inadvertently break established return
traffic. Applied in the zone policy matrix.

---

### ADR-006 — Traefik as Single Reverse Proxy Ingress
**Date:** April 2026
**Status:** Decided

**Context:**
Multiple Docker services need to be accessible via HTTPS without
exposing individual container ports. A reverse proxy is required.

**Decision:**
Traefik v3 is the single reverse proxy for all Docker services.
It handles TLS termination, Let's Encrypt certificate management via
Cloudflare DNS challenge, and routes requests to containers by Host
header. Traefik is the only container with ports bound to the host
network interface.

**Alternatives Considered:**
- Nginx Proxy Manager — rejected because Traefik's Docker provider
  auto-discovers containers via labels, eliminating manual route
  configuration for each new service
- Caddy — considered but Traefik's deeper integration with the Docker
  ecosystem and better support for complex middleware chains was
  preferred

**Consequences:**
New services are added to Traefik routing by adding labels to their
compose file — no manual Traefik configuration required. Wildcard
TLS cert via Let's Encrypt DNS challenge covers all subdomains.
Trade-off: Traefik configuration has a steeper initial learning curve
than Nginx Proxy Manager.

---

### ADR-007 — All Service Traffic Routed Through Traefik Only
**Date:** April 2026
**Status:** Decided

**Context:**
Docker containers could be accessed either directly by container port
or through Traefik. A consistent policy was needed.

**Decision:**
No container ports are exposed directly to any VLAN. All access to
all services goes through Traefik on port 443. This applies to both
internal clients and external traffic via Cloudflare Tunnel. Firewall
rules enforce this by only opening `obj_traefik:443` rather than
individual container ports.

**Alternatives Considered:**
- Allow direct port access for internal clients — rejected because
  it bypasses Traefik middleware (auth, rate limiting, headers) and
  creates inconsistency between internal and external access paths

**Consequences:**
Every service has consistent TLS, consistent middleware, and a single
choke point for access control. Firewall rules are simpler — one
destination IP and port covers all services. Trade-off: Traefik
becomes a critical single point of failure for all service access.

---

### ADR-008 — Docker on Dedicated Bare-Metal Host
**Date:** April 2026
**Status:** Decided

> ⚠️ VERIFY: This ADR's rationale cites "Intel Quick Sync transcoding
> (Jellyfin)," but ADR-028 states the host is a 2009 Xeon X3430 with no
> Quick Sync, and the E5-2630 v4 Proxmox host has no iGPU either. Either
> the hardware here is current (this rationale is wrong and bare-metal is
> justified on other grounds — VM overhead / direct I/O), or the host was
> upgraded (ADR-028 is stale). Confirm the current Docker host hardware
> and correct whichever ADR is wrong.

**Context:**
Docker services could run as VMs on Proxmox or on a dedicated
bare-metal server.

**Decision:**
Docker runs on a dedicated bare-metal host (EIRDOM-DOCKER-01) rather
than as a Proxmox VM. This provides direct hardware access for Intel
Quick Sync transcoding (Jellyfin), eliminates VM overhead for
network-intensive workloads, and isolates the media/web stack from
the hypervisor.

**Alternatives Considered:**
- Docker VM on Proxmox — rejected because GPU/iGPU passthrough for
  Quick Sync is more complex in a VM, and the workload profile
  (constant media transcoding, download client I/O) benefits from
  direct hardware access

**Consequences:**
Jellyfin hardware transcoding works without passthrough complexity.
Network throughput is not constrained by VM overhead. Trade-off:
an additional physical server to manage and power.

---

### ADR-009 — Proxmox VE for VM Hypervisor
**Date:** April 2026
**Status:** Decided

**Context:**
Windows Server VMs and Linux security monitoring VMs need a hypervisor.

**Decision:**
Proxmox VE 8.x is the hypervisor for all VMs. It provides a web-based
management UI, ZFS storage support, built-in backup tooling, and
VLAN-aware networking via Linux bridges.

**Alternatives Considered:**
- VMware ESXi — rejected due to Broadcom's acquisition changing the
  licensing model unfavorably for home use
- Windows Hyper-V — rejected because it would require Windows Server
  as the host OS, adding cost and complexity

**Consequences:**
Open-source hypervisor with no licensing cost. Strong community
support. Native ZFS for VM storage with snapshots and replication.
Trade-off: less polished UI than VMware for users coming from an
enterprise VMware background.

---

### ADR-010 — USW-Pro-Max-48-PoE for Both Switches
**Date:** April 2026
**Status:** Decided

**Context:**
The network requires two managed switches — one core in the garage
for house drops and APs, one distribution in the server room for
servers and security devices. Both need PoE++ for specific devices.

**Decision:**
Both switches are USW-Pro-Max-48-PoE units. Each provides 720W PoE
budget, 16x 2.5GbE ports, 16x PoE++ ports (60W each), 4x 10G SFP+
uplinks, and Etherlighting on all RJ45 ports.

**Alternatives Considered:**
- USW-Pro-48-PoE (previous generation) — rejected because it has only
  600W PoE budget, no 2.5GbE ports, and limited PoE++ port count
  which was a concern given the Access Hubs and U7 Pro Wall
- Mixed switch models (larger core, smaller distribution) — rejected
  for simplicity — identical hardware means identical management,
  identical spare parts, and identical firmware update procedures

**Consequences:**
720W per switch provides comfortable headroom (~37% core, ~14%
distribution). 2.5GbE ports give servers meaningful throughput
improvement over 1GbE without requiring 10GbE NICs. Etherlighting
provides visual VLAN identification at the rack. Trade-off: higher
cost than previous generation switches.

---

### ADR-011 — U7 Series Access Points
**Date:** April 2026
**Status:** Decided

**Context:**
The home needs Wi-Fi 7 coverage across interior rooms, the office,
exterior/garage, and bedroom wing.

**Decision:**
Deploy a mix of U7 Pro (main coverage), U7 Pro Wall (office in-wall),
U7 Pro Outdoor (garage/exterior), and U7 Lite (bedroom wing and
server room) access points. All are Wi-Fi 7 and managed by the
UDM-Pro-Max.

**Alternatives Considered:**
- Wi-Fi 6E APs (U6 series) — rejected because Wi-Fi 7 is the current
  generation and building a new home is the right time to deploy
  current-generation hardware
- Third-party APs — rejected because tight UniFi ecosystem integration
  (Etherlighting, fabric, controller management) is preferred

**Consequences:**
Wi-Fi 7 coverage across all areas including 6GHz band for high-speed
trusted devices. All APs managed from a single controller.
Trade-off: U7 Pro Wall requires 802.3bt PoE++ which limits port
placement to PoE++ capable ports on the core switch.

---

### ADR-012 — WPA3-Enterprise with 802.1X on Trusted SSID
**Date:** April 2026
**Status:** Decided

**Context:**
The primary trusted SSID needs strong authentication. Options ranged
from WPA3-Personal with a shared key to full WPA3-Enterprise with
802.1X certificate-based or credential-based authentication.

**Decision:**
The `Eirdom` SSID uses WPA3-Enterprise with 802.1X authenticated
via NPS on EIRDOM-DC-01. Domain-joined Windows devices authenticate
automatically via PEAP/MSCHAPv2 pushed by GPO. After PKI deployment,
machine certificates replace password-based EAP.

**Alternatives Considered:**
- WPA3-Personal with strong PSK — rejected because a shared key
  cannot be revoked per-device, and any compromised device or leaked
  key requires changing the password on all devices
- Open SSID with captive portal — rejected as completely inappropriate
  for a trusted network

**Consequences:**
Per-device, per-user authentication. Revoking a device's access
requires only disabling the AD account or machine certificate.
No shared secret to rotate. Trade-off: non-domain devices require
manual credential configuration.

---

### ADR-013 — Three-SSID Wireless Strategy
**Date:** April 2026
**Status:** Decided

**Context:**
Different device types have different trust levels and different
network access requirements. A strategy for SSID segmentation was
needed.

**Decision:**
Three SSIDs: `Eirdom` (VLAN 10, WPA3-Enterprise, trusted devices),
`Eirdom-IoT` (VLAN 20, WPA3-Personal, 2.4GHz only, client isolation),
`Eirdom-Guest` (VLAN 30, WPA3-Personal, internet only, client
isolation). IoT is 2.4GHz only to preserve 5/6GHz spectrum for
trusted and guest devices.

**Alternatives Considered:**
- Single SSID with VLAN assignment by device — rejected because UniFi
  dynamic VLAN assignment requires RADIUS and adds complexity without
  meaningful benefit over separate SSIDs
- Four SSIDs adding a separate IoT 5GHz SSID — rejected because most
  IoT devices do not support 5GHz and the additional SSID adds
  unnecessary beacon overhead

**Consequences:**
Clear network segmentation by device type. IoT devices are physically
incapable of reaching corporate resources. Guest devices are fully
isolated. Trade-off: 2.4GHz only for IoT means slightly reduced
throughput for the few IoT devices that do support 5GHz.

---

### ADR-014 — G6 Camera Fleet with AI Theta Interior
**Date:** April 2026
**Status:** Decided

**Context:**
The home needs comprehensive camera coverage including panoramic side
coverage, interior monitoring, front door, and garage.

**Decision:**
G6 180 (×4) for panoramic 180° coverage on house and garage sides,
G6 Bullet (×1) for garage interior, G6 Turret (×1) for front door
area, AI Theta (×1) for interior rooms, G4 Doorbell Pro PoE (×1) for
front door with integrated chime. All G6+ generation for consistency
and AI detection capability.

**Alternatives Considered:**
- G5 cameras — rejected because the build timeline allows for G6
  deployment and G6 provides a meaningful AI engine upgrade
- Third-party cameras — rejected because UniFi Protect ecosystem
  integration (AI detection, access control correlation, single app)
  is preferred over a mixed vendor environment

**Consequences:**
Unified management through UniFi Protect. AI detection (face, vehicle,
LPR) across all cameras. G6 180 eliminates coverage gaps on building
sides that would require multiple standard cameras. Trade-off: G6 180
requires 802.3at which slightly increases PoE budget vs standard
802.3af cameras.

---

### ADR-015 — All Cat6 Home-Runs to Garage Core Switch
**Date:** April 2026
**Status:** Decided

**Context:**
The home network requires wired drops throughout the house. The
garage houses the core switching infrastructure.

**Decision:**
All ethernet runs in the house are Cat6 home-runs terminating directly
at the core switch in the garage. No intermediate distribution switches
inside the house. Two-switch flat topology: core in garage, distribution
in garage server room.

**Alternatives Considered:**
- IDF closet inside the house with a secondary switch — rejected
  because the home's size does not justify the additional hardware,
  and home-runs to the garage are feasible with proper planning
  during construction

**Consequences:**
Simpler topology, fewer switches to manage, no intermediate switch
to power or cool inside the house. All cable management centralized
in the garage. Trade-off: longer cable runs for drops at the far end
of the house — must be verified against the 100m Cat6 maximum during
construction planning.

---

### ADR-016 — UFW Disabled in Favor of Docker iptables Management
**Date:** April 2026
**Status:** Decided

**Context:**
Ubuntu Server ships with UFW (Uncomplicated Firewall). Docker
manipulates iptables directly in the nat table, which means published
container ports bypass UFW rules entirely regardless of UFW
configuration.

**Decision:**
UFW is not enabled on EIRDOM-DOCKER-01. Docker manages iptables
directly. Host-level security is enforced by binding all containers
to `127.0.0.1` via `daemon.json` (`"ip": "127.0.0.1"`) and routing
all external traffic exclusively through Traefik.

**Alternatives Considered:**
- Enable UFW and attempt to work around Docker bypass — rejected
  because this creates a false sense of security where UFW rules
  appear to be enforcing policy but are actually being bypassed
- Use Docker's `--iptables=false` flag and manage iptables manually
  — rejected due to significant operational complexity

**Consequences:**
No false sense of security from UFW rules that Docker bypasses.
Container exposure is controlled by the `daemon.json` IP binding
and Traefik's port management. Trade-off: no host-level firewall
independent of Docker — security relies on correct compose
configuration and the network-level UDM-Pro-Max firewall.

---

### ADR-017 — Docker Daemon Bound to 127.0.0.1 by Default
**Date:** April 2026
**Status:** Decided

**Context:**
By default, Docker binds published container ports to `0.0.0.0`,
making them accessible from any network interface and any VLAN that
can reach the host. This is overly permissive for a production
environment.

**Decision:**
`/etc/docker/daemon.json` is configured with `"ip": "127.0.0.1"`.
This makes all published ports bind to localhost only by default.
The only exception is Traefik, which explicitly binds ports 80 and
443 to `0.0.0.0` in its compose file to accept external traffic.

**Alternatives Considered:**
- Leave default `0.0.0.0` binding — rejected because any VLAN that
  can reach EIRDOM-DOCKER-01 could access container ports directly,
  bypassing Traefik and firewall rules

**Consequences:**
No container except Traefik is reachable from the network even if a
firewall rule were misconfigured. Defense in depth — two independent
controls (firewall + daemon binding) both need to fail for a container
to be inadvertently exposed. Trade-off: easy to forget this setting
exists when troubleshooting connectivity issues.

---

### ADR-018 — IPS Enabled on All Security-Sensitive VLANs
**Date:** April 2026
**Status:** Decided

**Context:**
The UDM-Pro-Max includes a built-in Suricata IDS/IPS engine. A
decision was needed on which VLANs to enable it on and in which mode.

**Decision:**
IPS (block mode) enabled on VLANs 10, 20, 40, 50, and 60. IPS
disabled on VLAN 1 (Management — low risk, UniFi device traffic only)
and VLAN 30 (Guest — Cloudflare WAF provides upstream protection,
overhead not justified). Initial deployment in IDS mode for 1-2 weeks
before switching to block mode.

**Alternatives Considered:**
- IDS mode only (detect, never block) — rejected because detection
  without prevention only provides alerting value, not protection
- Enable on all VLANs including Management and Guest — rejected
  because the performance overhead on the gateway is not justified
  for those segments

**Consequences:**
Active blocking of known malware, C2 beacons, exploit attempts, and
other threats on all important segments. Trade-off: small performance
overhead on the UDM-Pro-Max and occasional false positives requiring
suppression during initial baselining.

---

### ADR-019 — GeoIP Blocking of 13 High-Risk Countries
**Date:** April 2026
**Status:** Decided

**Context:**
Inbound traffic from certain countries represents a disproportionate
share of scanning, ransomware, botnets, and state-sponsored attacks
based on observed threat intelligence patterns.

**Decision:**
Block inbound traffic from Russia, China, Iran, North Korea, Ukraine,
Belarus, Romania, Poland, Turkey, Pakistan, Nigeria, Nepal, and
Albania using UniFi's native GeoIP filtering. Reviewed quarterly.

**Alternatives Considered:**
- Block all countries except USA — rejected because it is overly
  aggressive and would block legitimate CDN traffic, software updates,
  and other services that route through international infrastructure
- No GeoIP blocking — rejected because it leaves a significant volume
  of known-bad source traffic reaching the WAN interface

**Consequences:**
Meaningful reduction in scanning and attack traffic reaching the WAN.
GeoIP blocking is acknowledged as an imperfect control — sophisticated
actors use infrastructure in non-blocked countries. It is one layer
in a defense-in-depth strategy, not a primary control. Trade-off:
legitimate traffic originating from or routing through blocked
countries may be dropped.

---

### ADR-020 — DoD IP Ranges Blocked at WAN
**Date:** April 2026
**Status:** Decided

**Context:**
US Department of Defense allocated IP ranges are frequently spoofed
in scanning and attack traffic despite being publicly routable
addresses. These ranges should never be appearing as attack sources
on a home network WAN.

**Decision:**
All 13 primary DoD /8 blocks published by ARIN are added to an IP
Group (`obj_dod_networks`) and blocked inbound at the WAN with
logging enabled. Reviewed quarterly against ARIN records.

**Alternatives Considered:**
- Trust DoD ranges — rejected because legitimate DoD traffic has no
  reason to initiate connections to a home network WAN interface

**Consequences:**
Spoofed DoD-sourced attack traffic is dropped at the WAN. Logging
provides visibility into how frequently these ranges are used in
attacks against the home IP. Trade-off: minimal — no legitimate use
case for DoD IP ranges initiating connections to a home network.

---

### ADR-021 — DHCP Relay to AD DC on VLAN 10, UDM Shadow Mode Elsewhere
**Date:** April 2026
**Status:** Decided

**Context:**
VLAN 10 (Corporate) needs DHCP from EIRDOM-DC-01 for full AD
integration (dynamic DNS registration, DHCP lease tracking in AD).
Other VLANs need DHCP but do not require AD integration.

**Decision:**
VLAN 10 uses DHCP relay to EIRDOM-DC-01 (10.1.10.10). All other
VLANs use the UDM-Pro-Max built-in DHCP server with Shadow Mode
enabled. Shadow Mode keeps the UniFi client list accurate even
when the UDM is not the authoritative DHCP server.

**Alternatives Considered:**
- UDM DHCP for all VLANs — rejected because VLAN 10 devices need
  AD-integrated DHCP for dynamic DNS registration and DHCP lease
  visibility in Active Directory Users and Computers
- AD DHCP for all VLANs — rejected because IoT, Guest, and Camera
  VLANs have no need for AD integration and it adds unnecessary
  dependency on EIRDOM-DC-01 availability for all DHCP

**Consequences:**
Corporate devices get full AD DHCP integration. Other VLANs get
simple UDM DHCP with accurate client tracking. Trade-off: two DHCP
servers to manage and monitor, and VLAN 10 DHCP is unavailable if
EIRDOM-DC-01 is offline.

---

### ADR-022 — Wazuh + Security Onion Dual Monitoring Stack
**Date:** April 2026
**Status:** Decided

**Context:**
The infrastructure needs security monitoring, log aggregation, and
network traffic analysis. A decision was needed on the tooling.

**Decision:**
Deploy both Wazuh (agent-based SIEM, log aggregation, compliance
scanning) and Security Onion 3.0 (network security monitoring,
passive traffic capture via SPAN). Both run as Proxmox VMs on VLAN 60.
Security Onion receives mirrored traffic via a SPAN port on the
distribution switch.

**Alternatives Considered:**
- Wazuh only — rejected because agent-based monitoring has blind spots
  for network-level threats that don't generate host logs
- Security Onion only — rejected because it lacks agent-based host
  monitoring and log aggregation
- Commercial SIEM — rejected due to cost and unnecessary complexity
  for a home environment

**Consequences:**
Two independent monitoring systems provide overlapping coverage.
Wazuh catches host-level events that Security Onion misses.
Security Onion catches network-level threats that Wazuh misses.
Trade-off: significant VM resource requirements (200GB+ storage for
Security Onion, 8GB+ RAM for Wazuh) and operational overhead
maintaining two monitoring platforms.

**Note (2026-07-01):** Wazuh agent enrollment requires BOTH the
`authd.pass` file AND `use_password yes` in `ossec.conf` — either alone
fails. Wazuh SSL cert for internal hosts must use DNS-01 (Cloudflare
plugin); HTTP-01 is structurally impossible for internal hosts. Planned
migration to internal PKI once all machines are domain-joined.

---

### ADR-023 — Private GitHub Repository for Infrastructure-as-Code
**Date:** April 2026
**Status:** Decided

**Context:**
Infrastructure configuration, Docker Compose files, scripts, and
documentation need version control, history, and a recovery path
if the server is rebuilt.

**Decision:**
All infrastructure configuration lives in a private GitHub repository
(this repo). Docker Compose files, scripts, documentation, and
`.env.example` files are committed. Actual `.env` files containing
secrets are gitignored. The repo is cloned to EIRDOM-DOCKER-01
for deployment.

**Alternatives Considered:**
- Local git repo without remote — rejected because it provides no
  off-site backup and no access from other devices
- Self-hosted Gitea — rejected because it adds a service dependency
  and the repo itself would need to be available before infrastructure
  can be recovered

**Consequences:**
Full version history for all configuration changes. Easy recovery
path — clone repo, run provision.sh, fill in .env files. Any device
can view and edit configuration. Trade-off: care must be taken to
never commit secrets — enforced by .gitignore and pre-commit hygiene.

---

### ADR-024 — Per-Service Docker Compose Files Over Single Monolithic File
**Date:** April 2026
**Status:** Decided

**Context:**
Docker services could be defined in a single `docker-compose.yml` or
split into per-service compose files.

**Decision:**
Each service group has its own `docker-compose.yml` in its own folder
(`docker/traefik/`, `docker/arr-stack/`, etc.). Services are started
independently with `docker compose -f docker/<service>/docker-compose.yml up -d`.

**Alternatives Considered:**
- Single monolithic compose file — rejected because updating one
  service requires touching a file that affects all services, and a
  syntax error in one service definition can prevent all services
  from starting

**Consequences:**
Services can be updated, restarted, and debugged independently.
The `update.sh` script handles startup ordering. Trade-off: slightly
more complex management compared to a single `docker compose up -d`
command, mitigated by the `update.sh` automation. See ADR-025 for the
env-file interpolation nuance that the Makefile workflow resolves.

---

### ADR-025 — Root .env for Shared Vars, Service .env for Specific Vars
**Date:** April 2026
**Status:** Decided

**Context:**
Docker Compose files need environment variables. Variables could all
live in one place or be split by scope.

**Decision:**
A root `.env` file holds variables shared across multiple services
(`TZ`, `PUID`, `PGID`, `UMASK`, `ROOT_DOMAIN`, `DOCKER_DATA_PATH`,
`MEDIA_PATH`, backup settings). Each service folder has its own `.env`
for service-specific variables. Compose files reference both via
`env_file: [../../.env, .env]` with the service file taking precedence
on conflicts.

**Alternatives Considered:**
- Single root .env for everything — rejected because it mixes concerns,
  makes it unclear which service uses which variable, and sensitive
  per-service credentials sit alongside generic shared config
- All vars in service .env files with duplication — rejected because
  shared values like TZ and PUID would need to be updated in every
  service file when changed

**Consequences:**
Clear separation of shared vs service-specific configuration. Changing
the timezone requires editing one file. Rotating a service API key
requires editing only that service's file. Trade-off: two files to
manage per deployment and the relative path `../../.env` must be
correct for each compose file location.

**Critical nuance (2026-07-01):** `env_file:` injects variables into the
CONTAINER only — it does NOT populate `${VAR}` interpolation in the compose
file itself. Interpolation reads only the `.env` in the working directory
or files passed via `--env-file`. A bare `docker compose up` therefore
leaves `${DOCKER_DATA_PATH}`/`${ROOT_DOMAIN}` blank, mounting volumes at
wrong paths. Resolved by the `docker/Makefile` orchestration, which always
passes `--env-file ../../.env` (and the local `--env-file .env` only when
the stack has one). Env changes require container RECREATION
(`make up SVC=<x>`), not `restart` — restart reuses the original container
env. This is operational detail; kept here as a consequence of the two-file
design rather than as a separate ADR.

---

### ADR-026 — Gluetun WireGuard VPN — qBittorrent-Only Scope
**Date:** 2026-04-12
**Status:** Decided

> ⚠️ VERIFY: Project history records ADR-042 as a TorGuard → AirVPN provider
> change, but ADR-042 is absent from this file and this ADR still names
> TorGuard as current. Confirm whether the VPN provider is now AirVPN; if so,
> this ADR should be marked "Superseded by ADR-042" and the provider-specific
> details (TorGuard, port forwarding path) updated. The Future-Decisions
> "TorGuard dedicated IP timing" item is likewise stale if AirVPN is in use.

**Context:**
Download traffic via qBittorrent exposes the home IP to torrent peers
and requires VPN protection. The question was how broadly to apply VPN
routing — all Docker traffic, only the download client, or something
in between.

**Decision:**
Only qBittorrent traffic is routed through the VPN via Gluetun. All
other ARR services (Radarr, Sonarr, Prowlarr etc.) use the normal
internet connection. qBittorrent shares Gluetun's network namespace
via `network_mode: service:gluetun` with a kill switch enforced.
TorGuard WireGuard protocol. Port forwarding enabled for better
seeding performance.

**Alternatives Considered:**
- Route entire Docker host through VPN — rejected because Cloudflare
  Tunnel, Authentik, and NetBox all need reliable unvpn'd outbound
  connections. A VPN failure would take down all services
- Route all arr-stack containers through VPN — rejected because Radarr
  and Prowlarr only make lightweight HTTPS metadata requests to public
  indexers — no torrent data flows through them
- OpenVPN instead of WireGuard — rejected due to higher CPU overhead
  on the already-constrained Xeon X3430

**Consequences:**
qBittorrent has no independent network interface — ARR apps must
connect to it at `http://gluetun:8080`, not `http://qbittorrent:8080`.
Traefik labels for the qBittorrent Web UI live on the Gluetun
container. Upgrading to a dedicated IP later requires only changing
`VPN_ENDPOINT_IP` in `.env`.

---

### ADR-027 — Dual Radarr + Dual Sonarr Instances for HD and 4K
**Date:** 2026-04-12
**Status:** Decided

**Context:**
The media library already contained separate HD and 4K content in
distinct folders (`radarr/`, `radarr-4k/`, `sonarr/`, `sonarr-4k/`).
Managing both qualities in a single Radarr/Sonarr instance requires
complex custom format scoring with risk of conflicts between HD and
4K quality profiles.

**Decision:**
Run two separate Radarr instances (radarr, radarr-4k) and two separate
Sonarr instances (sonarr, sonarr-4k). Each instance has its own root
folder, quality profile, download category, and Recyclarr
configuration. Prowlarr syncs indexers to all four instances.

**Alternatives Considered:**
- Single Radarr with custom format scoring — rejected because it
  creates fragile scoring logic and makes it hard to have different
  upgrade policies for HD vs 4K
- Single Radarr with multiple root folders — rejected because quality
  profiles still cannot be cleanly separated

**Consequences:**
Four ARR instances to maintain instead of two. Bazarr must connect to
all four. Jellyseerr connects to all four with separate quality
profiles. Recyclarr manages four separate instance configurations.
Trade-off is accepted — the separation is cleaner and more reliable.

---

### ADR-028 — Jellyfin Direct Play Only — No Transcoding
**Date:** 2026-04-12
**Status:** Decided

> ⚠️ VERIFY: This ADR states the host is a 2009 Xeon X3430 with no Quick Sync.
> ADR-008 justifies bare-metal by Intel Quick Sync. These conflict — confirm
> current Docker host hardware and correct whichever ADR is stale.

**Context:**
EIRDOM-DOCKER-01 runs an Intel Xeon X3430 (Nehalem, 2009) with no
Quick Sync, no VAAPI, and no hardware transcoding capability. The only
GPU present is a Silicon Motion SM712 display adapter with no media
acceleration support.

**Decision:**
Jellyfin is configured with transcoding disabled. All clients must
direct play content natively. 4K HDR and Dolby Vision content requires
a capable client (Apple TV 4K, Fire TV Stick 4K, Shield TV).

**Alternatives Considered:**
- Enable CPU transcoding — rejected because the 4-core 2.4GHz Xeon
  cannot transcode 4K HEVC in real time and would cause buffering
- Add a GPU — deferred. If hardware is upgraded to a system with Intel
  12th gen or newer iGPU, Quick Sync with HDR tone mapping can be
  enabled by uncommenting the `devices:` section in the compose file

**Consequences:**
Older or lower-capability clients (some browsers, older Smart TVs)
may fail to play certain content. Family members should use the
Jellyfin app on capable hardware. The `devices:` section is
intentionally left commented in the compose file as a reminder for
future hardware upgrades.

---

### ADR-029 — Authentik as SSO Provider — AD LDAP Source
**Date:** 2026-04-12
**Status:** Decided (updated 2026-07-01)

**Context:**
The infrastructure needed a centralized SSO solution so family members
log in once and access all internal services without per-service
accounts. Active Directory is already the identity store.

**Decision:**
Authentik provides SSO via Traefik ForwardAuth for browser-only internal
services and via native OIDC for apps that support it (see ADR-044 for the
full per-app integration model). Authentik syncs users and groups from AD
via LDAP against EIRDOM-DC-01 (see ADR-048 for source scope/filtering). No
Redis required — fully PostgreSQL-based since v2025.10. Services that manage
their own auth via LDAP (e.g. NetBox, Jellyfin) opt out of ForwardAuth using
`chain-public`/`chain-app` rather than `chain-standard`.

**Version note (2026-07-01):** Upgraded 2026.2 → 2026.5.3. Authentik upgrades
must be sequential by major version; 2026.2 → 2026.5 is adjacent and legal.
Targeting 2026.5.3 specifically clears the "session decode when upgrading from
2026.2" bug fixed in 2026.5.2. The 2026.5 default listen IP changed
`0.0.0.0` → `[::]`; if ForwardAuth 502s in an IPv4-only path, force
`AUTHENTIK_LISTEN__HTTP: "0.0.0.0:9000"` (and `__HTTPS`). Embedded outpost
upgrades atomically with the server image (no standalone outpost to sync).
2026.5 adds `core_default_app_access` to globally flip app access to
deny-by-default (see ADR-045).

**Alternatives Considered:**
- Keycloak — rejected due to significantly higher resource usage and
  complexity for a home lab use case
- Basic auth on Traefik — rejected because it cannot sync with AD and
  requires per-service credential management
- No SSO (per-service accounts) — rejected because it creates account
  sprawl and inconsistent security posture

**Consequences:**
All services behind `chain-standard` middleware require Authentik to
be healthy. If Authentik is down, those services become inaccessible.
OIDC apps depend on Authentik only at login — existing sessions survive
brief Authentik downtime. Traefik is the only service that must start
before Authentik. OIDC signing key must be RSA (not ECC); `email_verified`
default changed in Authentik 2025.10.

---

### ADR-030 — NetBox with netbox_unifi_sync Plugin — Custom Docker Image
**Date:** 2026-04-12
**Status:** Decided

**Context:**
The infrastructure needed a source of truth for IP address management,
device inventory, and network documentation. The `netbox_unifi_sync`
plugin was selected to automate populating NetBox from the UDM-Pro-Max
rather than maintaining it manually.

**Decision:**
NetBox v4.5 with the `netbox_unifi_sync` plugin installed via a custom
Dockerfile (`eirdom/netbox:v4.5`). The plugin must be installed in the
same image used by the netbox, netbox-worker, and netbox-housekeeping
containers. Authentication via AD LDAP directly — not Authentik
ForwardAuth — since NetBox has native LDAP support.

**Alternatives Considered:**
- Stock NetBox image + manual inventory — rejected because manual
  maintenance would quickly fall out of sync
- `mrzepa/unifi2netbox` external script — rejected in favour of the
  native NetBox plugin (`netbox_unifi_sync`) which runs inside the
  NetBox worker process and has a UI dashboard
- Nautobot — rejected due to higher complexity for this use case

**Consequences:**
Requires building a custom Docker image locally — not pulled from a
registry. `update.sh` uses `docker compose build --pull` for NetBox
instead of `docker compose pull`. The Dockerfile must be maintained
when upgrading NetBox base versions.

---

### ADR-031 — Single /media/arr Filesystem for All Media and Configs
**Date:** 2026-04-12
**Status:** Superseded by ADR-043

**Context:**
Docker hardlinks — which prevent duplicate disk usage during ARR
imports — only work within a single filesystem. The server has two
drives: a 100GB OS drive (sdb, LVM) and a 2.7TB media drive
(sda1, /media/arr, ext4).

**Decision:**
Both `DOCKER_DATA_PATH` (`/media/arr/config`) and `MEDIA_PATH`
(`/media/arr`) resolve to the same physical drive (sda1). All ARR
service configs, download directories, and media libraries live on
`/media/arr`. The OS drive (sdb) is used only for the Ubuntu system.

**Alternatives Considered:**
- Configs on OS drive, media on /media/arr — rejected because it
  places config and download directories on different filesystems,
  breaking hardlinks and causing full file copies on import
- NAS via NFS — deferred. A NAS could replace /media/arr in future
  if the drive needs to be expanded. NFS mounts can support hardlinks
  if the NFS server is configured correctly

**Consequences:**
If the 2.7TB drive fails, both configs and media are lost. Backup
strategy mitigates this: configs are backed up daily to NAS, media
is considered re-downloadable. The existing media library
(`radarr/`, `sonarr/` etc.) is preserved in place — no migration needed.

**Superseded (2026-06-26):** Replaced by the three-tier drive layout in
ADR-043, which separates configs (SSD), in-progress downloads (cache SSD),
and completed media + libraries (HDD) by access pattern while preserving
the hardlink invariant.

---

### ADR-032 — Uptime Kuma for Service Monitoring
**Date:** 2026-04-13
**Status:** Decided

**Context:**
With 15+ containers and several VMs running, there is no visibility
into service health beyond manually checking each URL. A monitoring
solution is needed so failures are detected proactively rather than
when a family member reports something is broken.

**Decision:**
Uptime Kuma deployed as a single Docker container on EIRDOM-DOCKER-01.
Monitors all services via HTTP, TCP, and Docker socket. Notification
channels configured for push/email alerts on downtime.

**Alternatives Considered:**
- Grafana + Prometheus — rejected as significantly more complex to
  set up and maintain for what is essentially uptime checking
- Healthchecks.io (cloud) — rejected because it would expose internal
  service URLs to an external service
- Manual checking — not viable at this scale

**Consequences:**
First service to know when something is down. The Docker socket mount
(read-only) gives Uptime Kuma container health visibility without
elevated privileges. Must configure notification channels immediately
after deployment — an unconfigured Uptime Kuma is useless.

---

### ADR-033 — Stirling PDF for Local PDF Processing
**Date:** 2026-04-13
**Status:** Decided

**Context:**
PDF manipulation tasks (merge, split, compress, OCR, convert) are
regularly needed for home administration — warranties, contracts,
architectural plans, tax documents. All current solutions require
uploading documents to third-party websites.

**Decision:**
Stirling PDF deployed as a stateless Docker container. No user data
is stored between sessions. All processing is in-memory. Replaces
all online PDF tools entirely — no documents leave the network.

**Alternatives Considered:**
- Online tools (SmallPDF, iLovePDF) — rejected because they upload
  sensitive documents to third-party servers
- LibreOffice CLI scripts — rejected as requiring per-task scripting
  with no web UI for family use

**Consequences:**
Fully stateless — no backup needed beyond the minimal config dir.
Max file size set to 100MB for large architectural drawings.

---

### ADR-034 — Paperless-ngx for Document Management
**Date:** 2026-04-13
**Status:** Decided

**Context:**
New home construction generates significant paperwork — warranties,
inspection reports, permits, HOA documents, insurance, contracts.
A searchable, OCR'd document archive is needed for long-term
home management.

**Decision:**
Paperless-ngx deployed with PostgreSQL and Redis on EIRDOM-DOCKER-01.
Authentik ForwardAuth provides SSO with header passthrough
(`X-Authentik-Username`) so users are auto-logged into Paperless
after Authentik authenticates them. Consume folder at
`${MEDIA_PATH}/paperless/consume/` for automatic document ingestion.

**Alternatives Considered:**
- Cloud document storage (Google Drive, OneDrive) — rejected because
  sensitive financial and legal documents should not live on third-party
  infrastructure
- Paperless without Authentik passthrough — rejected because requiring
  a separate login degrades the single-sign-on experience
- Mayan EDMS — rejected as significantly more complex for equivalent
  functionality

**Consequences:**
Initial admin credentials (`PAPERLESS_ADMIN_USER` and
`PAPERLESS_ADMIN_PASSWORD`) must be removed from `.env` after first
login to prevent re-creation on container restart. Document library
is backed up daily — both the PostgreSQL DB and the media directory.
Header passthrough requires `PAPERLESS_ENABLE_HTTP_REMOTE_USER=true`
(and the trusted-header/user-header settings) so the ForwardAuth header
is honored.

---

### ADR-035 — Immich for Family Photo and Video Backup
**Date:** 2026-04-13
**Status:** Decided (auth specifics: see ADR-046)

**Context:**
Family photos and videos are scattered across multiple iCloud and
Google Photos accounts. A self-hosted alternative is needed to
consolidate the family photo library on home infrastructure with
automatic mobile backup.

**Decision:**
Immich deployed with PostgreSQL (pgvecto.rs for ML vector search),
Redis, and a machine learning container for face recognition and CLIP
semantic search. Uses `chain-app` middleware (ADR-047) and authenticates
via native OIDC delegated to Authentik (ADR-046) — Immich mobile apps
cannot use Authentik ForwardAuth. Exposed externally via Cloudflare
Tunnel at `photos.eirdom.homes` for mobile backup away from home.

**Alternatives Considered:**
- PhotoPrism — rejected because Immich has superior mobile apps,
  faster active development, and better backup client support
- Nextcloud Photos — rejected as part of the broader Nextcloud
  rejection (ADR context — trying to do too much)
- Continued use of iCloud/Google Photos — rejected because it
  means family photos are on third-party infrastructure with no
  local copy

**Consequences:**
Requires pgvecto.rs PostgreSQL image rather than standard postgres —
this is the only service in the stack using a non-standard DB image.
ML models download on first start (~1GB) — allow extra time.
The photo library itself is not backed up by `backup.sh` — source
photos exist on family devices and the library is re-importable.
The PostgreSQL DB backup preserves all albums, faces, tags, and
metadata.

---

### ADR-036 — Ntfy for Self-Hosted Push Notifications
**Date:** 2026-04-13
**Status:** Decided

**Context:**
Multiple services (Uptime Kuma, Wazuh, backup.sh) need to deliver
alerts to phones and browsers. Relying on email introduces latency
and deliverability uncertainty. Third-party push services expose
alert content to external servers.

**Decision:**
Ntfy deployed as a lightweight single-container push notification
server. Services publish to topics via HTTP POST using access tokens.
Mobile apps subscribe to topics and receive instant push alerts.
Authentication is two-layer: Authentik ForwardAuth for the web UI,
ntfy token auth for service and mobile app connections.

**Alternatives Considered:**
- Gotify — rejected because Ntfy has better cross-platform support,
  a superior mobile app, and a simpler topic-based model
- Pushover (cloud) — rejected because alert content (server names,
  IPs, error details) should not leave the network
- Email only — rejected due to latency and missed alerts when email
  goes to spam

**Consequences:**
Each service that needs alerts must be configured with an ntfy topic
URL and access token after deployment. Mobile apps need the ntfy app
installed and configured to `https://ntfy.eirdom.homes`.

---

### ADR-037 — Homebox for Home Asset Inventory
**Date:** 2026-04-13
**Status:** Decided

**Context:**
New home construction generates a large number of new appliances,
tools, fixtures, and systems — each with warranty periods, serial
numbers, and service requirements. Without a tracking system, this
information gets lost within months of moving in.

**Decision:**
Homebox deployed as a single-container SQLite-backed inventory system.
Tracks all home assets with purchase dates, serial numbers, warranty
expiry, and linked documents. Pairs with Paperless-ngx for warranty
document storage.

**Alternatives Considered:**
- Spreadsheet — rejected because it has no reminders, no document
  linking, and no barcode scanning support
- Snipe-IT — rejected as significantly over-engineered for home use
  (designed for enterprise IT asset management)

**Consequences:**
Most valuable when populated from day one. Retroactively cataloguing
assets after moving in is significantly more work. Start immediately
when appliances are delivered during the build.

---

### ADR-038 — Grocy for Pantry and Grocery Management
**Date:** 2026-04-13
**Status:** Decided

**Context:**
A family home generates repetitive grocery shopping work — tracking
what's in stock, what's running low, and what's expired. A system
that tracks inventory and generates shopping lists from stock levels
reduces waste and simplifies weekly grocery runs.

**Decision:**
Grocy deployed as a single-container SQLite-backed ERP system for
the home. Handles pantry/fridge/freezer inventory with barcode
scanning, expiry tracking, shopping list generation, and household
task management. Uses linuxserver.io image for consistent PUID/PGID
handling.

**Alternatives Considered:**
- Mealie shopping lists only — rejected because Grocy's inventory
  tracking and barcode scanning are significantly more capable for
  actual pantry management
- OurGroceries (cloud app) — rejected because it requires a paid
  subscription and stores data externally

**Consequences:**
Default login is `admin`/`admin` — must be changed immediately on
first access. Grocy has a learning curve for initial setup of
products and stock locations. Mobile barcode scanning apps (Grocy
Android, Barcodebuddy) improve the daily workflow significantly.

---

### ADR-039 — Mealie for Recipe Management and Meal Planning
**Date:** 2026-04-13
**Status:** Decided

**Context:**
Family recipes are scattered across browser bookmarks, Pinterest,
physical cookbooks, and notes apps. A centralised recipe library
with meal planning and shopping list integration would simplify
weekly meal preparation and reduce the "what's for dinner" problem.

**Decision:**
Mealie deployed with PostgreSQL backend. Supports automatic recipe
import from any URL, weekly meal planning calendar, and shopping
list generation that integrates with Grocy. `ALLOW_SIGNUP` disabled —
admin invites family members. SMTP configured for meal plan
notification emails.

**Alternatives Considered:**
- Tandoor Recipes — rejected because Mealie has better active
  development, a superior URL import engine, and cleaner UX
- Nextcloud Cookbook — rejected as part of broader Nextcloud
  rejection (ADR context)
- Paprika (cloud app) — rejected because it stores recipes on
  third-party servers and has limited family sharing

**Consequences:**
Requires PostgreSQL unlike the simpler single-container services
in this batch. The `mealie-internal` Docker network isolates the
database. Initial recipe library population requires time investment
but URL import makes it fast for most online recipes. SMTP is shared
via the root `.env` SMTP block (see ADR-048 note / Gotchas re Resend).

---

### ADR-040 — Actual Budget for Personal Finance Tracking
**Date:** 2026-04-13
**Status:** Decided

**Context:**
Managing a new home build budget, mortgage, ongoing home expenses,
and family finances benefits from a dedicated budgeting tool. Cloud
financial services present a privacy concern as they require linking
bank accounts to external servers.

**Decision:**
Actual Budget deployed as a single-container SQLite-backed budgeting
server. Uses `chain-public` middleware — financial data requires
explicit login rather than Authentik header passthrough auto-login.
On first access a server password is set. Supports iOS, Android,
and desktop client apps. Zero cloud sync — all data stays local.

**Alternatives Considered:**
- YNAB (cloud) — rejected because it requires bank account linking
  to a third-party service and has an ongoing subscription cost
- Firefly III — rejected because Actual Budget has a significantly
  better UX, better mobile apps, and active development
- Spreadsheet — rejected because it lacks the zero-based budgeting
  workflow, reporting, and multi-account features needed

**Consequences:**
`chain-public` is intentional — financial data should never be
auto-logged in via Authentik header passthrough. The server password
is required once per new device/browser. Local-only means no
automatic bank import — transactions are entered manually or via
CSV import from bank statements.

---

### ADR-041 — Home Assistant OS on Proxmox VM for Smart Home
**Date:** 2026-04-13
**Status:** Decided

**Context:**
The new Eirdom home build requires a smart home hub to manage
automations, smart devices, cameras, and presence detection. UniFi
Protect cameras, door locks, lights, thermostats, and sensors all
need a central coordinator. The UniFi Fabric deployment prevents
creating local Network accounts, which eliminates the UniFi Network
integration for device presence tracking.

**Decision:**
Home Assistant OS (HAOS) deployed as a dedicated Proxmox VM (VM 140)
on VLAN 20 (IoT). HAOS is chosen over Docker because it includes the
Supervisor — which manages add-ons, updates, and backups — and because
it supports USB hardware passthrough (Zigbee/Z-Wave coordinators).

The UniFi Protect integration works via the Protect local API, which
is unaffected by the Fabric account restriction. The Network
integration (device presence) is replaced by GPS-based presence
detection via the HA mobile companion app.

`chain-public` middleware is used — the HA mobile companion app and
all device integrations cannot use Authentik ForwardAuth.

**Alternatives Considered:**
- Home Assistant Container (Docker) — rejected because it loses the
  Supervisor and therefore the entire add-on ecosystem
- Home Assistant Supervised on a VM — rejected as more complex to
  maintain than HAOS with equivalent functionality
- Skip Home Assistant entirely — rejected because the UniFi Protect
  integration provides significant value (motion events, doorbell
  triggers, camera feeds in dashboards) and smart home automation
  is a core goal for the new home build

**Consequences:**
HAOS is a separate VM from the main infrastructure stack — not managed
by Docker scripts. Updates are handled by HAOS itself (Settings →
System → Updates). Proxmox backup covers VM-level recovery. HAOS
internal backups cover configuration-level recovery. Two firewall
rules are required: DNS from 10.1.20.10 to EIRDOM-DC-01, and HTTPS
from 10.1.20.10 to Traefik (10.1.50.10).

---

> ⚠️ ADR-042 is referenced in project history as the VPN provider change
> (TorGuard → AirVPN) but is not present in this file. See the Verification
> Needed section. Either paste the original ADR-042 here or (re)write it, and
> update ADR-026 accordingly.

---

### ADR-043 — Three-Tier Drive Layout for the Media Stack
**Date:** 2026-06-26
**Status:** Decided — supersedes ADR-031

**Context:**
ADR-031 placed all media and Arr configs on a single filesystem to guarantee
hardlinks and atomic moves. It worked, but it sat two write-heavy,
latency-sensitive workloads — the Arr SQLite databases and qBittorrent's active
piece-writes/rechecks — on the same spinning disk as the bulk library. With dual
4K instances (ADR-027) and a 24/7 seeding profile, that concentrates random
writes on the worst-suited medium. Multiple physical drives are now available, so
workloads can be separated by access pattern.

**Decision:**
Split storage into three tiers by access pattern:
1. **Configs (Tier 1) → OS/Docker SSD.** All Arr + gluetun + qBittorrent config
   dirs (`${DOCKER_DATA_PATH}`). Small, random, latency-sensitive DB I/O on fast
   flash; backed up daily to NAS.
2. **In-progress downloads (Tier 2) → dedicated 250GB SSD.** qBittorrent's
   *incomplete* path only. Active piece-writes and rechecks hit flash. Self-
   clearing: completed torrents move off to Tier 3, keeping the cache free for new
   downloads — no cleanup script needed.
3. **Completed media + libraries (Tier 3) → 3TB WD Red.** qBittorrent's
   completed/seeding files AND all Arr libraries on one filesystem, mounted at
   `/data` in every relevant container. Write-once-read-many; preserves hardlinks.

**Hardlink invariant (load-bearing rule):** completed-downloads and library MUST
share one filesystem. qBittorrent and every *arr mount the Tier-3 root at the same
container path (`/data`), with `/data/downloads` (completed) and `/data/<app>`
(libraries) as subdirectories. The incomplete→completed transition (SSD → HDD) is
a cross-filesystem move (copy+delete), accepted as the cost of isolating active
writes; the library import that follows is an instant same-filesystem hardlink.

**Alternatives Considered:**

- Keep ADR-031 single filesystem — rejected: concentrates DB + active-download
  random writes on the media HDD. The original reason (hardlinks) is preserved by
  Tier 3 without forcing configs/active-downloads onto that disk.
- Completed downloads on the 250GB SSD, library on the 3TB — rejected: breaks
  hardlinks (separate filesystems), forcing a permanent duplicate per item and
  sizing the SSD for the whole seeding backlog instead of just active downloads.
- Download directly onto the 3TB, skip the cache SSD — rejected: returns active
  random piece-writes to the spinning disk, the exact thing the cache removes.

**Consequences:**

- qBittorrent gains a third mount (`/incomplete` → 250GB SSD); incomplete path set
  to `/incomplete`, default/completed save path to `/data/downloads`.
- `${DOCKER_DATA_PATH}` repoints from the media filesystem to the OS SSD; config
  dirs migrate there (stop stack, move dirs, restart).
- New path var for the cache drive, plus host mounts/fstab for the 250GB and 3TB,
  are prerequisites.
- Private trackers: completion-move clears the cache, but torrent removal must wait
  for required ratio/seed-time or you risk hit-and-run bans; set qBittorrent seed
  goals to the strictest tracker's rule.
- Single-drive Tier 3 has no redundancy; completed media is re-downloadable and
  config backups cover the DBs, so a Tier-3 failure means re-acquiring content,
  not data loss.
- 3TB is small for dual-4K remuxes. When Tier 3 grows (NAS/pool), the
  completed-downloads dir must move onto that same filesystem too, or hardlinks
  silently break again — the married pair (completed-downloads + library) migrates
  as a unit.
- Confirm the 3TB Red's model (CMR WD30EFRX vs SMR WD30EFAX); either works here,
  but if SMR, never add it to a ZFS pool or parity array.
- Supersedes ADR-031 — ADR-031's status set to "Superseded by ADR-043."

**Review Date:** when Tier 3 capacity is expanded (NAS/pool decision).

---

### ADR-044 — Application Authentication Integration Model (OIDC / ForwardAuth / Self-Auth)
**Date:** 2026-07-01
**Status:** Decided

**Context:**
With Authentik as the SSO provider (ADR-029), every app needs an
authentication integration path. Apps differ in what they support: some speak
OIDC natively, some can sit behind reverse-proxy ForwardAuth, some ship
mandatory built-in auth and can do neither cleanly. A single approach does not
fit all, and mismatches cause failures — an app doing its own OIDC placed
behind ForwardAuth double-authenticates, and mobile/native clients cannot
complete a browser ForwardAuth redirect.

**Decision:**
Classify each app into one of three integration tiers:
1. **Native OIDC (preferred where supported).** The app delegates login to an
   Authentik OAuth2/OpenID provider. True SSO, no separate password, works for
   mobile clients. Uses `chain-app` middleware (ADR-047). Immich is the
   reference (ADR-046).
2. **ForwardAuth (`chain-standard`).** Apps with no native SSO and browser-only
   access are protected by the embedded outpost via Traefik ForwardAuth.
   Uptime-Kuma is the reference.
3. **Self-auth (`chain-app`).** Apps with mandatory built-in auth that cannot do
   OIDC and whose clients cannot traverse ForwardAuth handle their own login;
   the proxy adds security headers only.

**Alternatives Considered:**
- ForwardAuth for everything — rejected: mobile/native clients cannot complete a
  browser redirect ForwardAuth flow, and apps with mandatory internal auth
  double-prompt.
- LDAP-direct for everything — rejected: not all apps support LDAP; OIDC via
  Authentik gives a consistent session/SSO layer and centralizes MFA/policy at
  Authentik rather than per-app binds.
- One tier only — rejected: no single mechanism fits all app capabilities.

**Consequences:**
Each new app's onboarding begins by choosing its tier, which also fixes its
Traefik middleware chain. OIDC apps get an OAuth2 provider + Application + group
binding (ADR-045) + app-side OIDC config. ForwardAuth apps hard-depend on
Authentik health; OIDC apps depend on Authentik only at login (existing sessions
survive brief downtime); self-auth apps are independent of Authentik. Trade-off:
three patterns to know instead of one — offset by each being the correct fit for
the app's capabilities.

---

### ADR-045 — Group-Based Application Authorization (Deny-by-Default via AD Group Bindings)
**Date:** 2026-07-01
**Status:** Decided

**Context:**
Beyond authentication (who you are), apps need authorization (whether you may use
this one). AD already contains purpose-built per-app and per-domain groups
(`Eirdom-Immich-Users`, `Eirdom-Jellyfin-Users`, `Eirdom-Family-Adults`, etc.)
synced into Authentik (ADR-048), with membership managed in AD.

**Decision:**
Authorize each Authentik Application by binding it to its matching
`Eirdom-<App>-Users` AD group via the Application's Policy/Group/User Bindings.
The instant a binding exists, the app is deny-by-default: only group members can
complete login (OIDC) or pass ForwardAuth, and only they see the dashboard tile.
Membership is managed exclusively in AD; Authentik enforces downstream.

**Alternatives Considered:**
- No bindings (Authentik default: open to any authenticated user) — rejected:
  every SSO user could reach every app; violates least privilege. (2026.5's
  `core_default_app_access` flag can flip the global default to deny-by-default —
  under review as a belt-and-suspenders default.)
- Per-app manual user assignment in Authentik — rejected: duplicates membership
  management outside AD and drifts from the AD source of truth.
- Custom expression policies per app — rejected: unnecessary when a 1:1 group
  already exists; reserved for conditional access (time/geo) later.

**Consequences:**
Access is granted by AD group membership alone — add a person to
`Eirdom-<App>-Users` in AD, sync, done. **Lockout risk: binding an app to a group
you are not in locks you out of that app.** Mitigate by adding yourself in AD and
syncing before binding; `akadmin` superuser bypasses bindings as the escape hatch.
Empty groups deny everyone, so verify membership synced before binding. Tile
visibility follows access for free (non-members do not see it); infra apps with no
group are hidden via the "Hide from Application Dashboard" toggle instead, since
they are plumbing, not user-facing. (Note: superusers may still see hidden/
unbound apps on their own dashboard as an admin affordance; per docs a hidden app
should not appear for regular users. Confirm behavior empirically with a non-admin
account.)

---

### ADR-046 — Immich Authentication via Native OIDC (Delegated to Authentik)
**Date:** 2026-07-01
**Status:** Decided — extends ADR-035

**Context:**
ADR-035 deployed Immich with "handles its own auth," unspecified. Immich supports
native OIDC and is used from mobile apps that cannot traverse ForwardAuth.

**Decision:**
Immich delegates login to Authentik via OIDC (ADR-044 tier 1). An Authentik
OAuth2/OpenID provider (Confidential client, RSA signing key, RS256) backs an
Application with slug `immich`. Immich's OAuth is configured with issuer
`https://auth.eirdom.homes/application/o/immich/`, scope `openid email profile`,
auto-register on. Access is gated to `Eirdom-Immich-Users` (ADR-045). Local
password login stays enabled as break-glass. Served at `photos.eirdom.homes`.

**Alternatives Considered:**
- Immich local accounts only — rejected: separate credentials per person, no SSO.
- ForwardAuth (`chain-standard`) — rejected: the Immich mobile app cannot complete
  a browser ForwardAuth redirect, and it would conflict with Immich's own session
  handling.
- LDAP — rejected: Immich has no first-class LDAP; OIDC is the supported SSO path.

**Consequences (includes hard-won setup constraints):**
- **Redirect URIs must match the real hostname exactly.** Authentik does exact-
  match. Register `https://photos.eirdom.homes/auth/login`,
  `https://photos.eirdom.homes/user-settings`, and `app.immich:///oauth-callback`
  (mobile). The `immich`-vs-`photos` hostname mismatch was the actual failure
  during setup — the issuer/slug stay `immich` (slug is hostname-independent), but
  the web redirect URIs use the real `photos` hostname. In 2026.5 set each URI's
  matching mode to Strict and type to Authorization.
- **Email claim keys the Immich user.** Only AD accounts with a populated `mail`
  attribute can log in; use real accounts (`tyler`, `irina`), not an admin-only
  account lacking a mailbox.
- **RSA signing key required** — ECC breaks RS256 discovery/verification.
- If Auto Launch is enabled later, `?autoLaunch=0` is the local-login escape.

---

### ADR-047 — `chain-app` Middleware for Self-Authenticating, Bursty Apps; Remove sslRedirect
**Date:** 2026-07-01
**Status:** Decided

**Context:**
Three middleware chains existed: `chain-standard` (authentik + security-headers),
`chain-public` (security-headers + rate-limit), `chain-admin` (authentik + ip
allowlist + security-headers). Placing Immich on `chain-public` made it hang:
Immich's bursty first paint (dozens of concurrent thumbnail/asset requests, plus
WebSockets) exceeded `rate-limit` (avg 100 / burst 50) → 429s → frontend retry
loop. Separately, `security-headers` carries `sslRedirect: true`, deprecated in
Traefik v3 and redundant because HTTPS is already forced at the `web`→`websecure`
entrypoint redirect.

**Decision:**
Add a fourth chain, `chain-app` = `security-headers` only (no ForwardAuth, no
rate-limit), for self-authenticating apps whose clients are bursty or non-browser
(Immich, Jellyfin). Remove `sslRedirect: true` from the `security-headers`
middleware.

**Alternatives Considered:**
- Keep Immich on `chain-public` — rejected: rate-limit breaks bursty media
  loading; rate-limiting a LAN media app is low value.
- No middleware on such apps — rejected: loses HSTS and other headers with no
  upside (auth is handled by the app / OIDC).
- Raise the rate-limit globally — rejected: masks the issue and weakens the limit
  where it is actually wanted (public/auth endpoints).

**Consequences:**
Media/self-auth apps get security headers without the auth or rate-limit that
break them. `rate-limit` stays where it belongs (public-facing/auth endpoints).
Removing `sslRedirect` eliminates a latent redirect-loop footgun with no loss —
the entrypoint still forces HTTPS. Trade-off: a fourth chain to understand.
(Middleware references from Docker labels must be suffixed `@file` when the
middleware is defined in the file provider — see Gotchas.)

---

### ADR-048 — AD LDAP Source Scope and Filtering
**Date:** 2026-07-01
**Status:** Decided — refines ADR-029

**Context:**
The AD LDAP source needs a base DN, a user filter, and a sync cadence. Naive
choices misfire: a domain-root base DN pulls ~80 built-in groups plus
Guest/krbtgt; an OU=Users-only base DN misses the groups; service accounts should
not become login-capable Authentik users.

**Decision:**
- **Base DN** `OU=Eirdom,DC=ad,DC=eirdom,DC=homes` — parent of both the Users and
  Groups OUs; pulls real users and the `Eirdom-*` groups without domain built-ins.
- **Service-account exclusion** via `memberOf` filter reusing the existing
  Windows-login-blocking group:
  `(&(objectClass=user)(!(objectClass=computer))(!(memberOf=CN=Eirdom-Service-Accounts,OU=Groups,OU=Eirdom,DC=ad,DC=eirdom,DC=homes)))`.
  One authoritative list — adding a service account to that group excludes it from
  Authentik automatically.
- **Sync cadence** `*/15 * * * *` (homelab-appropriate; the directory is tiny).
  Force an on-demand sync with `docker exec authentik-worker ak sync_ldap`.

**Alternatives Considered:**
- Domain-root base DN — rejected: imports built-in groups and system principals.
- OU=Users-only base DN — rejected: misses the `Eirdom-*` groups (they live in
  OU=Groups).
- Naming-convention filter (e.g. exclude `*-svc`) — rejected: brittle; the
  `memberOf` group is authoritative and self-maintaining.
- 2-hour default sync — acceptable but changed to 15 min so AD group changes
  reflect promptly.

**Consequences (includes hard-won constraints):**
- `memberOf` matches **direct** membership only. If service-account groups are ever
  nested, switch to the recursive rule
  `(!(memberOf:1.2.840.113556.1.4.1941:=CN=Eirdom-Service-Accounts,OU=Groups,OU=Eirdom,DC=ad,DC=eirdom,DC=homes))`.
- The group DN must be exact — the group object lives in OU=Groups even though its
  members live in OU=Service Accounts (a group's location and its members'
  locations are independent). A wrong DN fails silently.
- **LDAPS must connect by FQDN, not IP** (`ldaps://EIRDOM-DC-01.ad.eirdom.homes`) —
  the DC cert SAN contains only the hostname, so connecting by IP fails TLS
  validation.
- **Username mapping requires `list_flatten`** — AD returns attributes as arrays
  and the username field silently rejects a list. Custom source property mapping:
  `return {"username": list_flatten(ldap.get("sAMAccountName")), "name": list_flatten(ldap.get("displayName")), "email": list_flatten(ldap.get("mail"))}`.
- Group property mappings must use only the Name mapping (user-oriented default
  mappings applied to groups throw `Group() got unexpected keyword arguments`).
- Deleting a synced service-account *user entry* in Authentik is safe — the LDAP
  bind uses the AD account directly, not that entry.

---

## Implementation Gotchas — Recommend Relocating to Infrastructure Guide

The following are configuration gotchas discovered during deployment. They are
NOT architecture decisions (no meaningful alternatives), so they do not warrant
ADRs — forcing them into the ADR template would require inventing alternatives and
dilute the decision log. Recommended home: the Infrastructure Guide's
troubleshooting section (batched with the next guide revision). Listed here so
they are not lost pending that move.

- **Cloudflare DNS token env var:** the lego/Cloudflare provider reads
  `CF_DNS_API_TOKEN`, not `CF_API_TOKEN`. Traefik must have
  `CF_DNS_API_TOKEN: ${CF_API_TOKEN}` in its environment (saved in file, not just
  live) or the wildcard cert never issues.
- **`env_file` vs `${}` interpolation / `--env-file`:** covered as a consequence
  in ADR-025; the operational fix is the `docker/Makefile` always passing
  `--env-file ../../.env`. Env changes require `make up SVC=<x>` (recreate), not
  `restart`.
- **Traefik config mount paths:** static config must mount at
  `/etc/traefik/traefik.yml` and dynamic config at the file-provider directory
  (`/config/dynamic`); mounting elsewhere silently doesn't load.
- **Traefik v3 catch-all router syntax:** use `HostRegexp(\`^.+$\`)` (v3), not the
  v2 `HostRegexp(\`{host:.+}\`)`; the v2 form with high priority swallows every
  host.
- **File-provider middleware references from Docker labels need `@file`:** a bare
  middleware name on a Docker-labelled router resolves as `@docker`; middlewares
  defined in `middlewares.yml` (file provider) must be referenced as
  `<name>@file`. Middleware *definition* labels stay bare; only *references* get
  `@file`.
- **Resend SMTP:** `SMTP_USER` must be the literal string `resend` (not an email
  address); `SMTP_PASSWORD` is a Resend API key (`re_...`). Port 587 STARTTLS
  matches Authentik `USE_TLS: true`. Resend handles outbound; Cloudflare Email
  Routing handles inbound (MX vs TXT — no conflict).

---

## Future Decisions

Use the template at the top of this document to record decisions as
they are made during deployment and operation. Suggested topics:

- WordPress theme and plugin selection
- Wazuh alert tuning thresholds after baseline
- Security Onion ruleset selection after first week of captures
- UniFi Protect camera recording schedule and retention policy
- AD group structure expansion as more services are added
- Certificate template customization decisions
- NAS hardware selection and integration method (NFS vs SMB)
- UPS/battery backup strategy for server room
- VPN dedicated IP timing and port forwarding setup (⚠️ provider TBD pending
  ADR-042 verification — TorGuard vs AirVPN)
- Server hardware upgrade path (iGPU for Jellyfin transcoding — ⚠️ see ADR-008/028
  hardware verification)
- Immich external storage integration if the media tier fills up
- Grocy barcode scanner hardware selection
- Home Assistant Zigbee/Z-Wave coordinator hardware selection
- Smart device ecosystem decisions (Matter, Zigbee, Z-Wave)
- Remaining app OIDC/ForwardAuth onboarding per ADR-044 (Jellyfin plugin-based
  OIDC; Paperless HTTP remote-user; NetBox LDAP)