# Eirdom — Complete Deployment Guide
> Step-by-step deployment of the full Eirdom home infrastructure
> From bare hardware to fully operational system
> Last Updated: April 2026

---

## Before You Start

### What This Guide Covers

This guide deploys the complete Eirdom infrastructure from scratch in
the correct dependency order. Follow it top to bottom. Skipping phases
or reordering steps will cause failures that are hard to diagnose.

Estimated total deployment time: **2–3 days** for a first-time build.
Most time is waiting for Windows installs, AD replication, and cert
issuance — not active work.

### What You Need Before Starting

| Item | Details |
|------|---------|
| This repo cloned | On a workstation you'll work from |
| Password manager open | All generated credentials go here immediately |
| USB drive (×2) | For Root CA backup — encrypted, stored offsite |
| Windows Server 2025 ISO | ×3 copies needed (DC, Root CA, Sub CA) |
| Ubuntu 26.04 LTS ISO | For Wazuh VM |
| Security Onion 3.0 ISO | For Security Onion VM |
| Proxmox VE ISO | For the hypervisor |
| Cloudflare account | With `eirdom.homes` zone added |
| TorGuard account | WireGuard subscription, VPN credentials ready |
| Active internet connection | For package downloads during install |

### Credentials Policy

Every password, token, API key, and secret generated during this
deployment goes into your password manager **immediately** — before
moving to the next step. Nothing gets written on paper or in a text
file.

---

## IP Address Reference

Keep this table open throughout deployment.

| Host | IP | VLAN | Role |
|------|----|------|------|
| UDM-Pro-Max | 10.1.1.1 | 1 | Gateway / Firewall |
| USW-Core-01 | 10.1.1.2 | 1 | Core Switch (Garage) |
| USW-Dist-01 | 10.1.1.3 | 1 | Distribution Switch (Server Room) |
| EIRDOM-PVE-01 | 10.1.10.5 | 10 | Proxmox Hypervisor |
| EIRDOM-DC-01 | 10.1.10.10 | 10 | AD DC, DNS, DHCP, NPS |
| EIRDOM-SUB-01 | 10.1.10.12 | 10 | Issuing / Subordinate CA |
| EIRDOM-DOCKER-01 | 10.1.50.10 | 50 | Docker Host |
| EIRDOM-WAZUH-01 | 10.1.60.10 | 60 | Wazuh (on Proxmox VM 120) |
| EIRDOM-SONION-01 | 10.1.60.20 | 60 | Security Onion (on Proxmox VM 130) |

---

## Phase 0 — Prerequisites

### Step 0.1 — Clone the repo on your workstation

```bash
git clone git@github.com:YOUR_USER/eirdom.git
cd eirdom
```

### Step 0.2 — Prepare .env files

```bash
# Root .env
cp .env.example .env

# Service .env files
for dir in docker/*/; do
  [ -f "$dir/.env.example" ] && cp "$dir/.env.example" "$dir/.env"
done
```

Fill in `.env` with shared values — `ROOT_DOMAIN=eirdom.homes`,
`TZ=America/Chicago`, `PUID=1000`, `PGID=1000`. Leave service-specific
values for when you reach each service's deployment step.

### Step 0.3 — Cloudflare: Add eirdom.homes

1. Log into [dash.cloudflare.com](https://dash.cloudflare.com)
2. Add Site → enter `eirdom.homes` → Free plan
3. Update your domain registrar's nameservers to Cloudflare's
4. Wait for nameserver propagation (up to 24 hours — check with
   `dig NS eirdom.homes`)

### Step 0.4 — Cloudflare: Create API token for Traefik

1. Profile → API Tokens → Create Token
2. Template: **Edit zone DNS**
3. Permissions: Zone / DNS / Edit + Zone / Zone / Read
4. Zone Resources: Include → Specific zone → `eirdom.homes`
5. Save the token to your password manager

---

## Phase 1 — Physical Network (UniFi)

### Step 1.1 — Cable the switches

- UDM-Pro-Max → Core Switch (USW-Dist-01 in garage): 10G SFP+ DAC
- Core Switch → Distribution Switch (server room): 10G SFP+ DAC
- Plug EIRDOM-PVE-01 and EIRDOM-DOCKER-01 into the Distribution Switch

> Do not rack or cable Access Points, cameras, or door hubs yet —
> those come after wireless is configured in Phase 3.

### Step 1.2 — Initial UDM-Pro-Max setup

1. Connect a laptop directly to the UDM LAN port
2. Navigate to `unifi.ui.com` and adopt the UDM-Pro-Max
3. Set the UDM management IP: `10.1.1.1/24`
4. Set timezone: `America/Chicago`
5. Update firmware to latest before proceeding

### Step 1.3 — Create VLANs

In UniFi Network → Settings → Networks, create all 7 networks:

| Network Name | VLAN ID | Subnet | DHCP |
|-------------|---------|--------|------|
| Management | 1 | 10.1.1.0/24 | UDM (auto) |
| Corporate | 10 | 10.1.10.0/24 | **Relay** (configure after DC is up) |
| IoT | 20 | 10.1.20.0/24 | UDM |
| Guest | 30 | 10.1.30.0/24 | UDM |
| Cameras | 40 | 10.1.40.0/24 | UDM |
| Docker | 50 | 10.1.50.0/24 | UDM |
| Security | 60 | 10.1.60.0/24 | UDM |

Full configuration details in `unifi/vlans.md`.

### Step 1.4 — Create firewall zones and rules

Refer to `unifi/firewall/lan-rules.md` for the complete zone-based
firewall configuration. Implement all rules before any devices join
the network.

Key rules to verify:
- `DOCKER → CORPORATE`: DNS (TCP/UDP 53) + LDAP (TCP 389) to
  `obj_eirdom_dc01` — **required** for Authentik, NetBox, Jellyfin
- `SECURITY → ALL`: Allow all — Wazuh agents need to reach the manager
- `CAMERAS → ALL deny` except TCP 7443/7444 to UDM
- `GUEST → RFC1918 deny` — guests get internet only
- Zero WAN inbound rules — confirm no port forwards exist

### Step 1.5 — Configure switch port profiles

Follow `unifi/port-profiles.md` to configure port profiles on both
switches. Key profiles:

- **Trunk-All** — for Proxmox (carries all VLANs tagged)
- **VLAN-50** — for Docker host (untagged VLAN 50)
- **SPAN-Monitor** — for Security Onion monitor NIC (configured later)

Assign profiles to ports via UniFi Network → Devices → (switch) →
Ports.

---

## Phase 2 — Proxmox Hypervisor (EIRDOM-PVE-01)

### Step 2.1 — Install Proxmox VE

1. Download Proxmox VE ISO from `proxmox.com/downloads`
2. Flash to USB (Rufus or Etcher)
3. Boot EIRDOM-PVE-01 from USB
4. Installer settings:
   - Target disk: NVMe SSD (ZFS mirror if two drives available)
   - Management IP: `10.1.10.5/24`
   - Gateway: `10.1.10.1`
   - DNS: `1.1.1.1` (temporary — change to DC after AD is up)
   - Hostname: `eirdom-pve-01.ad.eirdom.homes`
5. Complete install and reboot

### Step 2.2 — Post-install Proxmox configuration

Access web UI at `https://10.1.10.5:8006`.

**Disable enterprise repository:**

```bash
# SSH to Proxmox
ssh root@10.1.10.5

# Disable enterprise repo (requires subscription)
echo "# disabled" > /etc/apt/sources.list.d/pve-enterprise.list

# Enable no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  >> /etc/apt/sources.list

apt update && apt full-upgrade -y
```

**Configure VLAN-aware bridge** in `/etc/network/interfaces`:

```
auto vmbr0
iface vmbr0 inet static
    address 10.1.10.5/24
    gateway 10.1.10.1
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

Apply with `ifreload -a` or reboot.

**Configure DNS A record** for Proxmox — add after DC is deployed:
```
proxmox.eirdom.homes    A    10.1.50.10   (points to Traefik)
```

---

## Phase 3 — Active Directory (EIRDOM-DC-01)

> AD is the authentication backbone. **Everything else depends on it.**
> Complete this phase fully before moving to Phase 4.

### Step 3.1 — Create and configure the DC VM

In Proxmox web UI, create VM 100:

| Setting | Value |
|---------|-------|
| VM ID | 100 |
| Name | EIRDOM-DC-01 |
| OS | Windows Server 2025 ISO |
| Machine | q35 |
| CPU | 4 cores, type: host |
| RAM | 8 GB (no ballooning) |
| Disk | 80 GB |
| Network | vmbr0, VLAN tag: 10 |

Install Windows Server 2025 with Desktop Experience. During install:
- No network needed — configure after
- Set local Administrator password → save to password manager

After install, configure network in Windows:
- IP: `10.1.10.10/24`
- Gateway: `10.1.10.1`
- DNS: `127.0.0.1`

Rename the computer to `EIRDOM-DC-01` and reboot.

### Step 3.2 — Install AD DS and promote to DC

In Server Manager → Add Roles and Features:
- Role: **Active Directory Domain Services**
- After install: click the flag notification → **Promote this server
  to a domain controller**

Promotion settings:
- Add a new forest
- Root domain name: `ad.eirdom.homes`
- Forest / Domain functional level: **Windows Server 2025**
- Check: **DNS Server** and **Global Catalog**
- Set DSRM password → save to password manager
- NetBIOS name: `EIRDOM`
- Accept default paths
- Install (server reboots automatically)

### Step 3.3 — Configure DNS forwarders

After reboot, open DNS Manager:
- Right-click server name → Properties → Forwarders
- Add: `1.1.1.1` and `9.9.9.9`

Create reverse lookup zone:
- DNS Manager → Reverse Lookup Zones → New Zone
- Active Directory-integrated, Primary
- Network ID: `10.1.10`

### Step 3.4 — Switch Proxmox DNS to DC

```bash
# On EIRDOM-PVE-01
nano /etc/resolv.conf
# Change nameserver to 10.1.10.10
```

### Step 3.5 — Configure DHCP

Install DHCP Server role from Server Manager. Authorize in AD.

Create scopes:

| Scope | Range | Gateway | DNS |
|-------|-------|---------|-----|
| VLAN10-Corporate | 10.1.10.100–200 | 10.1.10.1 | 10.1.10.10 |
| VLAN20-IoT | 10.1.20.100–200 | 10.1.20.1 | 10.1.10.10 |
| VLAN30-Guest | 10.1.30.100–200 | 10.1.30.1 | 1.1.1.1 |
| VLAN50-Docker | 10.1.50.100–200 | 10.1.50.1 | 10.1.10.10 |
| VLAN60-Security | 10.1.60.100–200 | 10.1.60.1 | 10.1.10.10 |

**Switch VLAN 10 DHCP relay in UniFi:**
Settings → Networks → Corporate → DHCP Mode → **DHCP Relay** →
Relay Server: `10.1.10.10`

### Step 3.6 — Create OU structure

In Active Directory Users and Computers, create the following OU tree
under the domain root:

```
ad.eirdom.homes
└── Eirdom
    ├── Users
    │   ├── Family
    │   └── Service Accounts
    ├── Computers
    │   ├── Workstations
    │   └── Servers
    └── Groups
        └── Security Groups
```

### Step 3.7 — Create user accounts

In `OU=Family,OU=Users,OU=Eirdom`:
- Create an account for each family member
- Set passwords → save to password manager
- All accounts: member of Domain Users

In `OU=Users,OU=Eirdom`:
- Create Tyler's admin account (separate from family account)
- Add to Domain Admins

### Step 3.8 — Create service accounts

In `OU=Service Accounts,OU=Users,OU=Eirdom`, create three accounts.
All with: password never expires, user cannot change password.

| Username | Purpose |
|----------|---------|
| `authentik-svc` | Authentik LDAP bind |
| `netbox-svc` | NetBox LDAP bind |
| `jellyfin-svc` | Jellyfin LDAP bind |

Save all passwords to password manager. These go into the respective
service `.env` files later.

### Step 3.9 — Create AD security groups

In `OU=Security Groups,OU=Groups,OU=Eirdom` — all Global Security:

| Group | Members |
|-------|---------|
| `Jellyfin-Users` | All family members |
| `Jellyfin-Admins` | Tyler (admin account) |
| `NetBox-Users` | Tyler |
| `NetBox-Staff` | Tyler |
| `NetBox-Admins` | Tyler |

Full reference: `docs/ad-groups.md`

### Step 3.10 — Install NPS and configure RADIUS

Install **Network Policy Server** role from Server Manager.

Register NPS in AD: in NPS console → right-click NPS → Register in AD.

Add RADIUS client:
- Friendly name: `UDM-Pro-Max`
- IP: `10.1.1.1`
- Shared secret: generate strong random string → save to password
  manager (needed in Step 3.11)
- Vendor: RADIUS Standard

Create Network Policy:
- Name: `Eirdom-Domain-WiFi`
- Conditions: Windows Group = `Domain Users` + NAS Port Type =
  Wireless IEEE 802.11
- Access: Grant
- EAP type: PEAP with MSCHAPv2

### Step 3.11 — Configure 802.1X on Eirdom SSID

> **Do not create the Eirdom SSID yet** — do this after PKI is
> deployed (Phase 4) so you can use a proper NPS certificate.
> For now, configure the RADIUS profile only.

In UniFi Network → Settings → Profiles → RADIUS:
- Add profile: `Eirdom-NPS`
- RADIUS IP: `10.1.10.10`
- Port: `1812`
- Shared secret: the secret from Step 3.10

### Step 3.12 — Create split-horizon DNS zone

This makes `*.eirdom.homes` resolve internally to Traefik (10.1.50.10)
rather than Cloudflare.

In DNS Manager on EIRDOM-DC-01:
- New Forward Lookup Zone
- Zone name: `eirdom.homes`
- AD-integrated
- Leave empty for now — add records in Step 7.4

### Step 3.13 — Configure base GPOs

In Group Policy Management, create and link these GPOs:

| GPO | Linked To | Key Settings |
|-----|-----------|-------------|
| Eirdom-Baseline-Security | Eirdom OU | Password: 12+ chars, 90-day max; lockout: 5 attempts / 30 min |
| Eirdom-Workstation-Config | Workstations OU | Windows Firewall enabled, BitLocker, 10-min screen lock |
| Eirdom-Wazuh-Agent | Computers OU | Configured after Wazuh is deployed (Phase 8) |
| Eirdom-Certificate-AutoEnroll | Eirdom OU | Configured after PKI is deployed (Phase 4) |

---

## Phase 4 — PKI (EIRDOM-RCA-01 + EIRDOM-SUB-01)

> The offline Root CA (VM 101) **must never be connected to a network.**
> Use the Proxmox console for all interaction. After issuing the
> subordinate certificate, shut it down immediately.

### Step 4.1 — Create Root CA VM (air-gapped)

Create VM 101 in Proxmox **with no network adapter**:

| Setting | Value |
|---------|-------|
| VM ID | 101 |
| Name | EIRDOM-RCA-01 |
| OS | Windows Server 2025 ISO |
| CPU | 2 cores |
| RAM | 4 GB |
| Disk | 40 GB |
| **Network** | **None — do not add a NIC** |

Install Windows Server 2025 (Desktop Experience). Do not join domain.
Rename to `EIRDOM-RCA-01`.

### Step 4.2 — Install and configure Root CA

In Server Manager → Add Roles → Active Directory Certificate Services:
- Role service: **Certification Authority only** (no web enrollment)
- Setup type: **Standalone CA** (not Enterprise — not domain-joined)
- CA type: **Root CA**
- New private key:
  - Algorithm: RSA
  - Key length: **4096**
  - Hash: **SHA-384**
- CA Common Name: `Eirdom Root CA`
- Validity period: **20 years**

After install, configure CRL and AIA distribution points via
Proxmox console:

```powershell
certutil -setreg CA\CRLPeriodUnits 52
certutil -setreg CA\CRLPeriod "Weeks"
certutil -setreg CA\CRLDeltaPeriodUnits 0
certutil -setreg CA\ValidityPeriodUnits 10
certutil -setreg CA\ValidityPeriod "Years"
Restart-Service certsvc
certutil -CRL
```

### Step 4.3 — Export Root CA cert and CRL to USB

Insert a USB drive via Proxmox console (VM → Hardware → Add → USB):

```powershell
copy C:\Windows\system32\CertSrv\CertEnroll\*.crt D:\
copy C:\Windows\system32\CertSrv\CertEnroll\*.crl D:\
```

**Shut down VM 101 immediately after copying.** Do not power it on
again until the annual CRL renewal. Back up the VM to two encrypted
USB drives and store offsite.

### Step 4.4 — Create Subordinate CA VM

Create VM 102 in Proxmox:

| Setting | Value |
|---------|-------|
| VM ID | 102 |
| Name | EIRDOM-SUB-01 |
| OS | Windows Server 2025 ISO |
| CPU | 2 cores |
| RAM | 4 GB |
| Disk | 60 GB |
| Network | vmbr0, VLAN tag: 10 |

Install Windows Server 2025 (Desktop Experience). Configure:
- IP: `10.1.10.12/24`, Gateway: `10.1.10.1`, DNS: `10.1.10.10`
- Join domain: `ad.eirdom.homes`
- Rename to `EIRDOM-SUB-01`

### Step 4.5 — Publish Root CA to AD

Copy the Root CA cert and CRL from USB to EIRDOM-SUB-01, then:

```powershell
certutil -dspublish -f "Eirdom Root CA.crt" RootCA
certutil -addstore -f root "Eirdom Root CA.crt"
certutil -addstore -f root "Eirdom Root CA.crl"
```

### Step 4.6 — Install Subordinate CA

Install IIS first (needed to host CRL/AIA files):

In Server Manager → Add Roles → Web Server (IIS). Configure a virtual
directory at `http://pki.eirdom.homes/CertEnroll/` pointing to
`C:\Windows\System32\CertSrv\CertEnroll\`.

Install AD CS role:
- Role services: Certification Authority + CA Web Enrollment
- Setup type: **Enterprise CA**
- CA type: **Subordinate CA**
- New private key: RSA 4096-bit, SHA-256
- CA Common Name: `Eirdom Issuing CA`
- Complete wizard — this generates a `.req` file

Copy the `.req` file to USB.

### Step 4.7 — Sign the subordinate certificate on Root CA

Power on VM 101 via Proxmox console. Insert USB:

```powershell
certreq -submit "EIRDOM-SUB-01.ad.eirdom.homes_Eirdom Issuing CA.req"
# Issue the pending request in CA MMC snap-in
certutil -ca.cert SubCA.cer
```

Copy `SubCA.cer` to USB. **Shut down VM 101 immediately.**

### Step 4.8 — Install the subordinate certificate

On EIRDOM-SUB-01:

```powershell
certutil -installcert SubCA.cer
Start-Service certsvc
```

Create DNS CNAME on EIRDOM-DC-01:
`pki.eirdom.homes → EIRDOM-SUB-01.ad.eirdom.homes`

### Step 4.9 — Configure certificate templates

In the CA MMC on EIRDOM-SUB-01, duplicate and configure:

| Template | Based On | Auto-Enroll | Purpose |
|----------|----------|-------------|---------|
| Eirdom-Computer | Computer | Yes | Machine auth, 802.1X |
| Eirdom-User | User | Yes | User auth |
| Eirdom-WebServer | Web Server | No | HTTPS for internal services |
| Eirdom-DomainController | DC Authentication | Yes | DC Kerberos/LDAPS |

### Step 4.10 — Enable certificate auto-enrollment GPO

Edit **Eirdom-Certificate-AutoEnroll** GPO:
- Computer Configuration → Policies → Windows Settings → Security
  Settings → Public Key Policies → Certificate Services Client -
  Auto-Enrollment
- Set to Enabled, check both renewal options

### Step 4.11 — Issue wildcard cert for Traefik

On EIRDOM-SUB-01 (or any domain-joined machine), request a cert using
the Eirdom-WebServer template with Subject Alternative Names:
- `*.eirdom.homes`
- `eirdom.homes`

Export as PFX, then convert to PEM for Traefik:

```bash
openssl pkcs12 -in wildcard.pfx -nocerts -nodes -out wildcard.key
openssl pkcs12 -in wildcard.pfx -clcerts -nokeys -out wildcard.crt
```

> **Note:** Traefik will also obtain its own wildcard cert via the
> Let's Encrypt DNS challenge for public-facing traffic. The internal
> PKI cert is for LAN clients. In practice, the Let's Encrypt cert
> covers both — the internal PKI cert can be deferred.

### Step 4.12 — Replace NPS certificate

On EIRDOM-DC-01, configure NPS to use the Eirdom-DomainController
certificate (auto-enrolled in Step 4.10) instead of the self-signed
one. In NPS → right-click server → Properties → scroll down to TLS
certificate.

### Step 4.13 — Create the Eirdom SSID (WPA3-Enterprise)

Now that PKI is deployed, create the trusted SSID in UniFi:

In Settings → WiFi → Create New WiFi:
- Network name: `Eirdom`
- Security: **WPA3 Enterprise**
- RADIUS Profile: `Eirdom-NPS` (from Step 3.11)
- VLAN: 10
- Band: 2.4 / 5 / 6 GHz
- All APs

Also create the remaining SSIDs:
- `Eirdom-IoT` — WPA3-Personal, VLAN 20, 2.4 GHz only
- `Eirdom-Guest` — WPA3-Personal, VLAN 30, 2.4+5 GHz

Full SSID settings in `unifi/wireless.md`.

---

## Phase 5 — Docker Host (EIRDOM-DOCKER-01)

### Step 5.1 — Install Ubuntu 26.04 LTS

EIRDOM-DOCKER-01 is a bare-metal server, not a VM.

Boot from Ubuntu 26.04 LTS Server USB:
- Static IP: `10.1.50.10/24`
- Gateway: `10.1.50.1`
- DNS: `10.1.10.10`
- Hostname: `eirdom-docker-01`
- Username: `dockeradmin`
- Install OpenSSH: yes
- No snaps

### Step 5.2 — Verify /media/arr is mounted

The 2.7TB data drive (sda1) must be mounted at `/media/arr`:

```bash
lsblk
# Should show sda1 mounted at /media/arr

# If not, add to /etc/fstab:
echo "UUID=$(blkid -s UUID -o value /dev/sda1) /media/arr ext4 defaults 0 2" \
  | sudo tee -a /etc/fstab
sudo mount -a
```

### Step 5.3 — Join the AD domain

```bash
sudo apt install -y sssd realmd adcli krb5-user samba-common-bin
sudo realm join ad.eirdom.homes -U Administrator
```

### Step 5.4 — Run provision.sh

Clone the repo onto the Docker host and run provisioning:

```bash
git clone git@github.com:YOUR_USER/eirdom.git /opt/eirdom
cd /opt/eirdom

# Copy your filled-in .env files from your workstation
# OR fill them in now
cp .env.example .env
nano .env

sudo bash scripts/provision.sh
```

`provision.sh` handles:
- Docker Engine installation
- daemon.json configuration (bind to 127.0.0.1)
- Directory structure creation under `/media/arr/config/` and `/media/arr/`
- Docker network creation (proxy, wordpress-internal, authentik-internal,
  netbox-internal, arr-internal, jellyfin-internal, monitoring)
- NetBox custom image build
- All service `.env.example` → `.env` copies

---

## Phase 6 — Traefik + Cloudflare Tunnel

> Traefik must be the first Docker service started.
> Everything else depends on it being healthy.

### Step 6.1 — Fill in Traefik .env

```bash
cd /opt/eirdom/docker/traefik
nano .env
```

Required values:
- `TRAEFIK_ACME_EMAIL` — your email for Let's Encrypt
- `CF_API_TOKEN` — Cloudflare API token from Step 0.4
- `CF_ZONE_ID` — from Cloudflare dashboard → eirdom.homes → Zone ID

### Step 6.2 — Start Traefik

```bash
cd /opt/eirdom/docker/traefik
docker compose up -d
docker compose logs -f
```

Watch for `Certificate obtained successfully` — takes 30–90 seconds.

### Step 6.3 — Create the Cloudflare Tunnel

1. Cloudflare dashboard → Zero Trust → Networks → Tunnels
2. Create tunnel → Name: `Eirdom-Tunnel`
3. Select Docker → copy the tunnel token → save to password manager
4. Note the Tunnel ID (UUID)

Fill in Cloudflared .env:

```bash
cd /opt/eirdom/docker/cloudflared
nano .env
# CLOUDFLARE_TUNNEL_TOKEN=<token from above>
# CLOUDFLARE_TUNNEL_ID=<UUID from above>
```

Configure public hostnames in Zero Trust dashboard:

| Subdomain | Domain | Service |
|-----------|--------|---------|
| `@` | `eirdom.homes` | `http://traefik:80` |
| `www` | `eirdom.homes` | `http://traefik:80` |
| `jellyfin` | `eirdom.homes` | `http://traefik:80` |
| `requests` | `eirdom.homes` | `http://traefik:80` |
| `photos` | `eirdom.homes` | `http://traefik:80` |

Start Cloudflared:

```bash
cd /opt/eirdom/docker/cloudflared
docker compose up -d
docker compose logs -f
# Watch for: "Tunnel connected"
```

### Step 6.4 — Add internal DNS records

On EIRDOM-DC-01, in DNS Manager → Forward Lookup Zones →
`eirdom.homes` → add the following A records (all pointing to
`10.1.50.10`):

```
eirdom.homes        A   10.1.50.10
www                 A   10.1.50.10
traefik             A   10.1.50.10
auth                A   10.1.50.10
netbox              A   10.1.50.10
jellyfin            A   10.1.50.10
requests            A   10.1.50.10
jellystat           A   10.1.50.10
radarr              A   10.1.50.10
radarr4k            A   10.1.50.10
sonarr              A   10.1.50.10
sonarr4k            A   10.1.50.10
lidarr              A   10.1.50.10
prowlarr            A   10.1.50.10
bazarr              A   10.1.50.10
qbit                A   10.1.50.10
status              A   10.1.50.10
pdf                 A   10.1.50.10
paperless           A   10.1.50.10
photos              A   10.1.50.10
ntfy                A   10.1.50.10
homebox             A   10.1.50.10
grocy               A   10.1.50.10
mealie              A   10.1.50.10
actual              A   10.1.50.10
homeassistant       A   10.1.50.10
proxmox             A   10.1.50.10
wazuh               A   10.1.50.10
securityonion       A   10.1.50.10
```

And separately (pointing directly to their hosts):

```
pki    CNAME   EIRDOM-SUB-01.ad.eirdom.homes
```

---

## Phase 7 — Authentik SSO

### Step 7.1 — Fill in Authentik .env

```bash
cd /opt/eirdom/docker/authentik
nano .env
```

Key values:
- `AUTHENTIK_SECRET_KEY` — generate with
  `openssl rand -base64 50` — **save immediately, never change**
- `AUTHENTIK_DB_PASSWORD` — strong random password
- `AUTHENTIK_LDAP_BIND_PASSWORD` — password for `authentik-svc` AD
  account from Step 3.8

### Step 7.2 — Start Authentik

```bash
cd /opt/eirdom/docker/authentik
docker compose up -d
docker compose logs -f authentik-server
```

Wait for `Successfully started Authentik` in logs (2–3 minutes).

### Step 7.3 — Complete initial Authentik setup

Navigate to `https://auth.eirdom.homes/if/flow/initial-setup/`

Set the `akadmin` password → save to password manager.

### Step 7.4 — Configure AD LDAP source

Full setup in `docker/authentik/README.md`. Summary:

1. Admin UI → Directory → Federation & Social Login → Add LDAP Source
2. Server URI: `ldap://10.1.10.10:389`
3. Bind DN: `CN=authentik-svc,OU=Service Accounts,OU=Eirdom,DC=ad,DC=eirdom,DC=homes`
4. Bind password: `authentik-svc` password
5. Base DN: `DC=ad,DC=eirdom,DC=homes`
6. User search: `OU=Eirdom,DC=ad,DC=eirdom,DC=homes`
7. Run sync — verify AD users appear under Directory → Users

### Step 7.5 — Configure Traefik ForwardAuth provider

In Authentik admin:
1. Applications → Providers → Create → Proxy Provider
2. Name: `Traefik ForwardAuth`
3. Forward auth (single application) → External host: `https://auth.eirdom.homes`
4. Create Application linked to this provider
5. Applications → Outposts → Edit embedded outpost → add the provider
6. Verify outpost URL is `https://auth.eirdom.homes`

---

## Phase 8 — WordPress (Family Website)

### Step 8.1 — Fill in webserver .env

```bash
cd /opt/eirdom/docker/webserver
nano .env
# WP_DB_ROOT_PASSWORD, WP_DB_PASSWORD — strong random passwords
```

### Step 8.2 — Start WordPress

```bash
cd /opt/eirdom/docker/webserver
docker compose up -d
```

Navigate to `https://eirdom.homes` — complete WordPress setup wizard.

Install recommended plugins: Wordfence (security), disable XML-RPC.

---

## Phase 9 — Media Stack (ARR + Jellyfin)

### Step 9.1 — Fill in arr-stack .env

```bash
cd /opt/eirdom/docker/arr-stack
nano .env
```

Key values from TorGuard account:
- `WIREGUARD_PRIVATE_KEY`
- `WIREGUARD_ADDRESSES`
- `VPN_ENDPOINT_IP` / `VPN_ENDPOINT_PORT`
- `WIREGUARD_PUBLIC_KEY`

Leave API keys blank for now — you'll fill these in after the containers
start and generate their own keys.

### Step 9.2 — Create Recyclarr secrets file

```bash
cp /opt/eirdom/docker/arr-stack/secrets.yml.example \
   /media/arr/config/recyclarr/secrets.yml
# Fill in API keys after containers start
```

### Step 9.3 — Start the ARR stack

```bash
cd /opt/eirdom/docker/arr-stack
docker compose up -d

# Watch Gluetun connect first
docker logs gluetun --follow
# Wait for: "VPN server ip..."  and "TUN device..." messages
```

### Step 9.4 — Configure ARR apps

After containers are up, configure each app. Access via:
- Radarr HD: `https://radarr.eirdom.homes`
- Radarr 4K: `https://radarr4k.eirdom.homes`
- Sonarr HD: `https://sonarr.eirdom.homes`
- Sonarr 4K: `https://sonarr4k.eirdom.homes`
- Lidarr: `https://lidarr.eirdom.homes`
- Prowlarr: `https://prowlarr.eirdom.homes`
- qBittorrent: `https://qbit.eirdom.homes`

**In Prowlarr:**
1. Add indexers (all of them here — Prowlarr syncs to ARR apps)
2. Settings → Apps → Add each ARR app
3. Add FlareSolverr: Settings → Indexers → FlareSolverr →
   URL: `http://flaresolverr:8191`

**In Radarr HD, Radarr 4K, Sonarr HD, Sonarr 4K:**
1. Settings → Download Clients → Add qBittorrent
   - Host: `gluetun` (**not** `qbittorrent`)
   - Port: `8080`
2. Settings → Media Management → enable **Use Hardlinks**
3. Set root folder to appropriate `/data/<library>/` path
4. Note the API key (Settings → General) → save to password manager

**In Recyclarr** — fill in `secrets.yml`:

```bash
nano /media/arr/config/recyclarr/secrets.yml
# Add API keys for all 4 ARR instances
```

Run initial sync:

```bash
docker exec recyclarr recyclarr sync
```

### Step 9.5 — Fill in jellyfin .env

```bash
cd /opt/eirdom/docker/jellyfin
nano .env
# JELLYSTAT_DB_PASSWORD, JELLYSTAT_JWT_SECRET
# JELLYFIN_LDAP_BIND_PASSWORD (jellyfin-svc password from Step 3.8)
# JELLYFIN_API_KEY — fill in after Jellyfin starts
```

### Step 9.6 — Start Jellyfin stack

```bash
cd /opt/eirdom/docker/jellyfin
docker compose up -d
```

### Step 9.7 — Configure Jellyfin

Navigate to `https://jellyfin.eirdom.homes` — complete setup wizard:

1. Add media libraries pointing to:
   - Movies (HD): `/data/radarr/`
   - Movies (4K): `/data/radarr-4k/`
   - TV (HD): `/data/sonarr/`
   - TV (4K): `/data/sonarr-4k/`
   - Music: `/data/lidarr/`

2. Install LDAP Authentication plugin:
   - Admin → Plugins → Catalog → LDAP Authentication → Install
   - Restart Jellyfin
   - Configure LDAP: server `10.1.10.10`, bind user `jellyfin-svc`,
     user filter: `(&(objectClass=user)(memberOf=CN=Jellyfin-Users...))`
   - See `docker/jellyfin/README.md` for full DN strings

3. Disable transcoding:
   - Admin → Playback → Transcoding → uncheck all transcoding options

4. Note API key (Admin → API Keys → +) → save to password manager →
   add to `jellyfin/.env` as `JELLYFIN_API_KEY`

### Step 9.8 — Configure Jellyseerr

Navigate to `https://requests.eirdom.homes`:
1. Sign in with Jellyfin account
2. Connect to Jellyfin server: `http://jellyfin:8096`
3. Add Radarr instances (HD and 4K), Sonarr instances (HD and 4K)

### Step 9.9 — Configure Bazarr

Navigate to `https://bazarr.eirdom.homes`:
1. Connect to all four ARR instances
2. Set subtitle providers and languages

---

## Phase 10 — NetBox (Network IPAM)

Full setup guide: `docker/netbox/README.md`

### Step 10.1 — Create UniFi read-only account

In UniFi Network → Settings → Admins & Users:
- Username: `netbox-sync`
- Role: View Only
- Save password to password manager

### Step 10.2 — Fill in NetBox .env

```bash
cd /opt/eirdom/docker/netbox
nano .env
# NETBOX_DB_PASSWORD, NETBOX_REDIS_PASSWORD, NETBOX_SECRET_KEY
# NETBOX_LDAP_BIND_PASSWORD (netbox-svc password from Step 3.8)
# UNIFI_USERNAME=netbox-sync, UNIFI_PASSWORD=<from above>
```

### Step 10.3 — Start NetBox

```bash
cd /opt/eirdom/docker/netbox
docker compose up -d
docker compose logs -f netbox
# Wait for: "Starting development server"
```

### Step 10.4 — Initial NetBox configuration

Navigate to `https://netbox.eirdom.homes`:
1. Log in with AD credentials (NetBox-Admins group member)
2. Plugins → UniFi Sync → Configure controller:
   - URL: `https://10.1.1.1`
   - Username: `netbox-sync`
   - Password: from password manager
3. Run initial sync dry-run → review → run live sync
4. Import Ubiquiti device type library:
   ```bash
   docker exec -it netbox /opt/netbox/venv/bin/python \
     /opt/netbox/netbox/manage.py import_device_types \
     --vendor Ubiquiti
   ```

---

## Phase 11 — Security Monitoring (Wazuh + Security Onion)

### Step 11.1 — Deploy EIRDOM-WAZUH-01

Full guide: `docs/wazuh-setup.md`. Summary:

1. Create VM 120 in Proxmox (Ubuntu 24.04, 4 vCPU, 8GB RAM, 200GB,
   VLAN 60, IP 10.1.60.10)
2. Install Ubuntu, configure static IP
3. Run Wazuh all-in-one installer:
   ```bash
   curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
   sudo bash wazuh-install.sh -a
   ```
4. Save admin credentials to password manager
5. Verify at `https://wazuh.eirdom.homes` (via Traefik)

### Step 11.2 — Configure UDM syslog to Wazuh

UniFi Network → Settings → System → Logging → Remote Logging:
- Server IP: `10.1.60.10`
- Port: `514`
- Protocol: UDP

### Step 11.3 — Deploy Wazuh agents

Full guide: `docs/wazuh-agents.md`. Deploy to all endpoints:

| Endpoint | Method | Group |
|----------|--------|-------|
| EIRDOM-DC-01 | Manual MSI install | windows-servers |
| EIRDOM-SUB-01 | Manual MSI install | windows-servers |
| EIRDOM-DOCKER-01 | APT install | linux-servers |
| EIRDOM-PVE-01 | APT install | linux-servers |
| Family workstations | GPO (auto on domain join) | windows-workstations |

### Step 11.4 — Deploy EIRDOM-SONION-01

1. Create VM 130 in Proxmox with **two NICs**:
   - NIC 1: vmbr0, VLAN 60 (management — `10.1.60.20`)
   - NIC 2: vmbr0, no VLAN tag, **promiscuous** (monitor — no IP)
2. Boot Security Onion 3.0 ISO → Standalone install
3. Assign NIC 1 as management, NIC 2 as monitor
4. Set management IP: `10.1.60.20/24`

### Step 11.5 — Configure port mirroring for Security Onion

In UniFi Network → Devices → Distribution Switch → Settings →
Port Mirroring:
- Mirror source: UDM-Pro-Max uplink port
- Mirror destination: port connected to Security Onion monitor NIC

Verify in Security Onion → Grid → (node) that packets are arriving.

### Step 11.6 — Configure Wazuh GPO deployment

Now that Wazuh is running, activate the Wazuh GPO:

Edit **Eirdom-Wazuh-Agent** GPO (linked to Computers OU):
- Startup script:
  ```batch
  msiexec /i \\EIRDOM-DC-01\NETLOGON\wazuh-agent.msi /q
    WAZUH_MANAGER="10.1.60.10"
    WAZUH_REGISTRATION_SERVER="10.1.60.10"
    WAZUH_AGENT_GROUP="windows-workstations"
  ```

### Step 11.7 — Connect Wazuh to Ntfy

> **Prerequisite:** Complete Phase 16 Step 16.1 (Ntfy) first so you
> have a service token to use here.

This routes critical Wazuh alerts to your phone. Full instructions
in `docs/wazuh-setup.md` Phase 6 — summary:

1. Create `/var/ossec/integrations/custom-ntfy` on EIRDOM-WAZUH-01
   with the script from the Wazuh setup guide
2. Replace `YOUR_NTFY_TOKEN_HERE` with your ntfy service token
3. Add the `<integration>` block to `/var/ossec/etc/ossec.conf`
4. Restart: `sudo systemctl restart wazuh-manager`
5. Verify a test notification arrives on your phone

---

## Phase 12 — NetOps Toolkit

### Step 12.1 — Set up Python environment

```bash
cd /opt/eirdom/tools/netops-toolkit
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Step 12.2 — Fill in config

```bash
cp config/config.example.yaml config/config.yaml
nano config/config.yaml
# Fill in UDM credentials, Anthropic API key, output paths
```

### Step 12.3 — Test collection

```bash
python scripts/unifi_collector.py --target health
```

Should return UDM health data. If successful, run full collection:

```bash
python scripts/unifi_collector.py --target all
```

### Step 12.4 — Set up cron job

```bash
crontab -e
```

Add (replacing `/opt/eirdom` with actual path if different):

```cron
# Weekly network snapshot — Sunday 2 AM
0 2 * * 0 cd /opt/eirdom/tools/netops-toolkit \
  && /opt/eirdom/tools/netops-toolkit/venv/bin/python \
     scripts/collect_and_document.py \
  >> /var/log/eirdom-netops.log 2>&1
```

Also add backup cron:

```cron
# Daily backup — 2 AM
0 2 * * * cd /opt/eirdom && sudo bash scripts/backup.sh
```

Full cron schedule in `docs/cron.md`.

---

## Phase 13 — Access Points, Cameras, and Access Control

### Step 13.1 — Adopt Access Points

With the Eirdom, Eirdom-IoT, and Eirdom-Guest SSIDs now configured,
power on and adopt each AP:

- 2× U7 Pro (hallway near master/hall bath; living room ceiling)
- 1× U7 Pro Wall (working office)
- 1× U7 Pro Outdoor (garage exterior)
- 2× U7 Lite (hallway between bed 2/3; server room)

After adoption, assign each AP to the appropriate AP group and verify
all three SSIDs are broadcasting.

### Step 13.2 — Adopt cameras

Connect cameras to the network (VLAN 40) and adopt in UniFi Protect:

- G5 Pro × 2 (front entry, driveway)
- G6 Instant 180 × 3 (left/right/rear exterior)
- AI Theta × 3 (living room, kitchen/dining, garage interior)
- G5 Dome × 1 (server room)

Configure recording schedules and retention in Protect.

### Step 13.3 — Configure Access Control

Adopt UA-Hubs and Reader Pro G2 units in UniFi Access:
- Front door
- Laundry entry
- Rear door
- Server room

Configure access policies — Tyler full access, family members
appropriate zones.

---

## Phase 15 — Home Services (Uptime Kuma, Stirling PDF, Paperless, Immich)

### Step 15.1 — Start Uptime Kuma

```bash
cd /opt/eirdom/docker/uptime-kuma
docker compose up -d
```

Navigate to `https://status.eirdom.homes` and complete first-time
setup — create an admin account and save credentials to password
manager. Then add monitors for every service:

| Monitor Type | Name | URL / Target |
|-------------|------|-------------|
| HTTP(s) | WordPress | `https://eirdom.homes` |
| HTTP(s) | Traefik | `https://traefik.eirdom.homes` |
| HTTP(s) | Jellyfin | `https://jellyfin.eirdom.homes/health` |
| HTTP(s) | Radarr | `https://radarr.eirdom.homes` |
| HTTP(s) | Sonarr | `https://sonarr.eirdom.homes` |
| HTTP(s) | Prowlarr | `https://prowlarr.eirdom.homes` |
| HTTP(s) | Authentik | `https://auth.eirdom.homes` |
| HTTP(s) | NetBox | `https://netbox.eirdom.homes` |
| HTTP(s) | Paperless | `https://paperless.eirdom.homes` |
| HTTP(s) | Immich | `https://photos.eirdom.homes` |
| HTTP(s) | Wazuh | `https://wazuh.eirdom.homes` |
| TCP | EIRDOM-DC-01 DNS | `10.1.10.10` port `53` |
| TCP | EIRDOM-DC-01 LDAP | `10.1.10.10` port `389` |

Configure a notification channel (Settings → Notifications) — email
via SMTP or a push notification service — so downtime actually alerts
you rather than sitting silently in a dashboard.

### Step 15.2 — Start Stirling PDF

```bash
cd /opt/eirdom/docker/stirling-pdf
docker compose up -d
```

No further setup required. Navigate to `https://pdf.eirdom.homes`
and verify the UI loads. Stirling PDF is ready to use immediately.

### Step 15.3 — Fill in Paperless .env and start

```bash
cd /opt/eirdom/docker/paperless
nano .env
# Fill in: PAPERLESS_DB_PASSWORD, PAPERLESS_SECRET_KEY,
#          PAPERLESS_ADMIN_PASSWORD
```

Generate values:
```bash
openssl rand -base64 32  # DB password
openssl rand -base64 50  # secret key
```

Start the stack:

```bash
docker compose up -d
docker compose logs -f paperless
# Wait for: "Startup successful"
```

Navigate to `https://paperless.eirdom.homes`. You will be
automatically logged in as the Authentik-authenticated user via
the `X-Authentik-Username` header passthrough — no separate
Paperless login needed after Authentik authenticates you.

**After first successful login:**

Remove the initial admin credentials from `.env` so they are
not reused on container restart:

```bash
# Edit docker/paperless/.env
# Remove or blank out:
# PAPERLESS_ADMIN_USER=
# PAPERLESS_ADMIN_PASSWORD=
docker compose up -d  # restart to apply
```

**Set up the document inbox:**

The consume folder at `${MEDIA_PATH}/paperless/consume/` is
watched automatically. Any PDF or image dropped here is ingested,
OCR'd, and indexed within seconds. Point a network scanner,
iOS Files app, or any other source at this path.

### Step 15.4 — Fill in Immich .env and start

```bash
cd /opt/eirdom/docker/immich
nano .env
# Fill in: IMMICH_DB_PASSWORD
```

```bash
docker compose up -d
docker compose logs -f immich-server
# Wait for: "Immich Server is listening on..."
# Note: first start takes longer — ML models download (~1GB)
```

Navigate to `https://photos.eirdom.homes`. The first account
you create becomes the admin — **create your account before
sharing the URL with family.**

**Invite family members:**

Administration → Users → Create User for each family member.
They will receive an email invite and set their own password.

**Configure the mobile app:**

Share these instructions with family (or add to `docs/family-setup.md`):

1. Download **Immich** from the App Store or Google Play
2. Server URL: `https://photos.eirdom.homes`
3. Log in with the email and password from the invite
4. Settings → Background Backup → Enable
5. Choose backup frequency and WiFi-only options

**Add `photos` to the Cloudflare Tunnel public hostnames** if not
already done in Step 6.3:

Zero Trust → Networks → Tunnels → Eirdom-Tunnel → Public Hostnames:
- Subdomain: `photos`, Domain: `eirdom.homes`, Service: `http://traefik:80`

---

## Phase 14 — Verification Checklist

Work through this list before considering the deployment complete.

### Network

- [ ] External portscan shows zero open ports on home IP
      (`nmap -Pn YOUR_HOME_IP` from outside — expect all filtered)
- [ ] `eirdom.homes` resolves publicly to Cloudflare proxy IPs
      (never your home IP)
- [ ] `jellyfin.eirdom.homes` resolves internally to `10.1.50.10`
- [ ] IoT device cannot ping `10.1.10.1` (VLAN isolation working)
- [ ] Guest device cannot ping any RFC1918 address
- [ ] Camera VLAN has no internet (except TCP 7443/7444 to UDM)

### Authentication

- [ ] Domain user can log into Eirdom WiFi with WPA3-Enterprise
- [ ] AD user in `Jellyfin-Users` can log into Jellyfin
- [ ] AD user in `NetBox-Users` can log into NetBox
- [ ] Authentik SSO redirects correctly for Radarr, Prowlarr etc.
- [ ] `chain-admin` services (Traefik, Wazuh) refuse access from
      VLAN 20/30

### Services

- [ ] `https://eirdom.homes` — WordPress loads publicly
- [ ] `https://traefik.eirdom.homes` — Traefik dashboard loads
- [ ] `https://jellyfin.eirdom.homes` — Jellyfin loads, media visible
- [ ] `https://requests.eirdom.homes` — Jellyseerr loads
- [ ] Radarr can reach qBittorrent via `gluetun:8080`
- [ ] Recyclarr sync successful: `docker exec recyclarr recyclarr sync`
- [ ] VPN is active: `docker exec gluetun wget -qO- ifconfig.io`
      (should return a TorGuard IP, not your home IP)
- [ ] NetBox shows UniFi devices after sync
- [ ] `https://wazuh.eirdom.homes` — Wazuh dashboard loads
- [ ] Wazuh shows all expected agents as Active
- [ ] `https://status.eirdom.homes` — Uptime Kuma loads, all monitors green
- [ ] `https://pdf.eirdom.homes` — Stirling PDF loads
- [ ] `https://paperless.eirdom.homes` — Paperless loads, Authentik auto-login works
- [ ] Drop a PDF into `${MEDIA_PATH}/paperless/consume/` — verify it ingests
- [ ] `https://photos.eirdom.homes` — Immich loads
- [ ] Immich mobile app connects and backup runs on a test device
- [ ] `https://ntfy.eirdom.homes` — Ntfy loads
- [ ] Push notification received on phone from Uptime Kuma test alert
- [ ] `https://homebox.eirdom.homes` — Homebox loads
- [ ] `https://grocy.eirdom.homes` — Grocy loads, default password changed
- [ ] `https://mealie.eirdom.homes` — Mealie loads, recipe import works
- [ ] `https://actual.eirdom.homes` — Actual Budget loads, server password set

## Phase 16 — Productivity Services (Ntfy, Homebox, Grocy, Mealie, Actual)

### Step 16.1 — Start Ntfy

```bash
cd /opt/eirdom/docker/ntfy
docker compose up -d
```

Create users and generate access tokens:

```bash
# Create admin user
docker exec ntfy ntfy user add --role=admin tyler

# Create a read-only subscriber for family mobile apps
docker exec ntfy ntfy user add family

# Generate access token for services (Uptime Kuma, Wazuh etc.)
docker exec ntfy ntfy token add tyler
# Save the token to password manager
```

**Configure Uptime Kuma notifications:**

Settings → Notifications → Add Notification:
- Type: ntfy
- Server URL: `https://ntfy.eirdom.homes`
- Topic: `eirdom-alerts`
- Token: the token generated above

**Mobile app setup:**
Install ntfy → Add server → `https://ntfy.eirdom.homes` →
authenticate → subscribe to `eirdom-alerts` and `eirdom-backup`.

**Enable backup.sh notifications:**

Add your ntfy token to the root `.env` on EIRDOM-DOCKER-01:

```bash
nano /opt/eirdom/.env
# Set: NTFY_TOKEN=tk_your_token_here
```

`backup.sh` reads `NTFY_TOKEN` automatically and posts to the
`eirdom-backup` topic on success and failure. No further changes
needed — it is already wired in.

**Connect Wazuh → Ntfy (do after Step 11.7):**

Once you have a service token, return to Step 11.7 and complete
the Wazuh integration. Subscribe to `eirdom-security` in the
ntfy mobile app to receive security alerts.

### Step 16.2 — Start Homebox

```bash
cd /opt/eirdom/docker/homebox
docker compose up -d
```

Navigate to `https://homebox.eirdom.homes`. Create an admin account
on first visit. Start adding assets as appliances arrive during the
build — model numbers, serial numbers, purchase dates, warranty
expiry, and links to warranty documents in Paperless.

### Step 16.3 — Start Grocy

```bash
cd /opt/eirdom/docker/grocy
docker compose up -d
```

Navigate to `https://grocy.eirdom.homes`. Default login is
`admin` / `admin` — **change immediately** under User Settings.

Initial setup: configure locations (Pantry, Fridge, Freezer),
product groups, and your household units under Settings.

### Step 16.4 — Fill in Mealie .env and start

```bash
cd /opt/eirdom/docker/mealie
nano .env
# Fill in: MEALIE_DB_PASSWORD, MEALIE_ADMIN_PASSWORD
docker compose up -d
docker compose logs -f mealie
# Wait for: "Application startup complete"
```

Navigate to `https://mealie.eirdom.homes`. Log in with the admin
credentials. Invite family members via Admin → Manage Users.

### Step 16.5 — Start Actual Budget

```bash
cd /opt/eirdom/docker/actual
docker compose up -d
```

Navigate to `https://actual.eirdom.homes`. Set the server password
on first access — save to password manager. Create your first
budget file and configure accounts.

**Desktop and mobile apps:** point to `https://actual.eirdom.homes`
and enter the server password once per device.

---

### Backups

- [ ] `sudo bash scripts/backup.sh` runs without errors
- [ ] Backup directory created at `${BACKUP_PATH}`
- [ ] VM 101 (Root CA) backup stored on two encrypted USB drives,
      one offsite

---

## Phase 17 — Home Assistant

Full guide: `docs/homeassistant-setup.md`

### Step 17.1 — Deploy EIRDOM-HAOS-01

1. Download the HAOS QCOW2 image on EIRDOM-PVE-01
2. Create VM 140 (2 vCPU, 4 GB RAM, 32 GB disk, VLAN 20)
3. Import and attach the HAOS disk image
4. Start the VM and complete onboarding at `http://10.1.20.10:8123`
5. Set static IP `10.1.20.10/24`, gateway `10.1.20.1`, DNS `10.1.10.10`

### Step 17.2 — Add Traefik route

Add the `homeassistant` router and `homeassistant-svc` service to
`docker/traefik/dynamic/routers.yml` — see `docs/homeassistant-setup.md`
Phase 3 for the exact YAML blocks. Traefik picks up the change
without a restart.

Add DNS record on EIRDOM-DC-01:
```
homeassistant    A    10.1.50.10
```

### Step 17.3 — Add firewall rules

In UniFi Network, add two rules to the IOT zone (before
`iot_corporate_deny`):
- `ha_corp_dns` — `10.1.20.10` → `10.1.10.10` TCP/UDP 53 Allow
- `ha_docker_traefik` — `10.1.20.10` → `10.1.50.10` TCP 443 Allow

### Step 17.4 — Configure integrations

1. **UniFi Protect** — add via Settings → Integrations, host `10.1.1.1`
2. **Mobile companion app** — install on all family phones, point to
   `https://homeassistant.eirdom.homes`
3. **Jellyfin** — add via Settings → Integrations
4. **Ntfy** — configure for automation push notifications

---

## What To Do After First Deployment

With everything running, these are the next operational priorities:

1. **Wazuh baseline** — let Wazuh run for 1–2 weeks before tuning
   alert thresholds. The first days will be noisy.

2. **Recyclarr tuning** — review quality profiles after first
   downloads. Adjust custom format scores in `recyclarr.yml` to taste.

3. **WordPress** — install theme, create family content, configure
   family member accounts.

4. **PKI auto-enrollment** — verify domain-joined workstations are
   receiving certificates from EIRDOM-SUB-01 automatically.

5. **Security Onion tuning** — review Suricata ruleset and suppress
   known-good internal traffic signatures.

6. **Annual reminders** — set calendar reminders for:
   - Root CA CRL renewal (annually)
   - `eirdom.homes` domain registration renewal
   - Full infrastructure documentation review

---

## Related Documentation

| Document | When To Use |
|----------|------------|
| [`docs/services.md`](services.md) | Service URLs, ports, auth for every service |
| [`docs/network-diagram.md`](network-diagram.md) | Traffic flow reference |
| [`docs/decisions.md`](decisions.md) | Why things are built the way they are |
| [`docs/ad-groups.md`](ad-groups.md) | AD group and service account reference |
| [`docs/wazuh-setup.md`](wazuh-setup.md) | Wazuh VM deployment detail |
| [`docs/wazuh-agents.md`](wazuh-agents.md) | Wazuh agent deployment detail |
| [`docs/cron.md`](cron.md) | All scheduled tasks |
| [`docs/family-setup.md`](family-setup.md) | Non-technical family guide |
| [`unifi/vlans.md`](../unifi/vlans.md) | VLAN configuration detail |
| [`unifi/wireless.md`](../unifi/wireless.md) | SSID configuration detail |
| [`unifi/firewall/lan-rules.md`](../unifi/firewall/lan-rules.md) | Firewall rules detail |
| [`docker/authentik/README.md`](../docker/authentik/README.md) | Authentik setup detail |
| [`docker/arr-stack/README.md`](../docker/arr-stack/README.md) | ARR + Jellyfin setup detail |
| [`docker/netbox/README.md`](../docker/netbox/README.md) | NetBox setup detail |
| [`docker/traefik/README.md`](../docker/traefik/README.md) | Traefik setup detail |