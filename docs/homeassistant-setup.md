# Home Assistant OS — Deployment Guide
> EIRDOM-HAOS-01 · VM 140 · VLAN 20 · 10.1.20.10
> Home Assistant Operating System (HAOS)
> Last Updated: April 2026

---

## Overview

Home Assistant is the smart home hub for Eirdom. It runs on a dedicated
Proxmox VM using Home Assistant Operating System (HAOS) — not Docker.

**Why HAOS on a VM, not Docker:**
- HAOS includes the Supervisor, which manages add-ons, updates, and
  backups. Running HA in Docker (without the Supervisor) loses the
  entire add-on ecosystem
- HAOS gets direct access to USB hardware (Zigbee/Z-Wave coordinators,
  USB sensors) via Proxmox USB passthrough
- HAOS is fully isolated from the Docker stack

**Sits on VLAN 20 (IoT)** so it can reach all smart devices directly.
Accessible at `https://homeassistant.eirdom.homes` via Traefik.

**Authentication:** Home Assistant handles its own auth. `chain-public`
middleware is used — the HA mobile companion app and integrations
cannot use Authentik ForwardAuth.

---

## VM Specification

| Field | Value |
|-------|-------|
| VM ID | 140 |
| Hostname | `EIRDOM-HAOS-01` |
| OS | Home Assistant OS (HAOS) |
| vCPU | 2 (minimum) — 4 recommended |
| RAM | 4 GB (minimum) — 8 GB recommended |
| Disk | 32 GB |
| Network | VLAN 20 (IoT) |
| IP | 10.1.20.10/24 (static) |
| Gateway | 10.1.20.1 |
| DNS | 10.1.10.10 (EIRDOM-DC-01) |
| Internal URL | `https://homeassistant.eirdom.homes` |

> Home Assistant is not resource-heavy. 2 vCPU and 4 GB RAM is
> comfortable for a typical home with 50–100 devices. Add RAM later
> via Proxmox if needed.

---

## Phase 1 — Prepare the Proxmox VM

### Step 1 — Download the HAOS QCOW2 image

HAOS is distributed as a pre-built disk image — not an ISO. Download
from the Proxmox web UI shell or via SSH:

```bash
# SSH to EIRDOM-PVE-01 or use the Proxmox web shell
cd /var/lib/vz/images

# Check the latest version at:
# https://github.com/home-assistant/operating-system/releases
# Replace X.X with the latest stable version number

wget -O haos.qcow2.xz \
  "https://github.com/home-assistant/operating-system/releases/latest/download/haos_ova-$(curl -s https://api.github.com/repos/home-assistant/operating-system/releases/latest | grep tag_name | cut -d '"' -f4 | tr -d 'v').qcow2.xz"

unxz haos.qcow2.xz
mv haos.qcow2 haos_vm140.qcow2
```

### Step 2 — Create the VM in Proxmox

In Proxmox web UI → Create VM:

**General:**
- VM ID: `140`
- Name: `EIRDOM-HAOS-01`

**OS:**
- Do not use any media — select "Do not use any media"
- Guest OS Type: Linux, Version: 6.x - 2.6 Kernel

**System:**
- Machine: `q35`
- BIOS: `OVMF (UEFI)`
- Add EFI Disk: checked (small disk, default storage)
- SCSI Controller: `VirtIO SCSI single`
- Qemu Agent: checked

**Disks:**
- **Delete** the default disk that Proxmox creates — the HAOS image
  replaces it

**CPU:**
- Sockets: 1
- Cores: 2
- Type: `host`

**Memory:**
- 4096 MB (4 GB)
- Ballooning: disabled

**Network:**
- Bridge: `vmbr0`
- VLAN Tag: `20`
- Model: `VirtIO`

Click **Finish** but **do not start** yet.

### Step 3 — Import the HAOS disk image

```bash
# Run on EIRDOM-PVE-01 (SSH or web shell)
qm importdisk 140 /var/lib/vz/images/haos_vm140.qcow2 local-lvm

# The above imports the disk. Now attach it to the VM:
# In Proxmox web UI → VM 140 → Hardware → Unused Disk 0
# → Edit → Bus/Device: SCSI 0 → Add
```

Alternatively via CLI:

```bash
# After importdisk, attach it
qm set 140 --scsi0 local-lvm:vm-140-disk-1
qm set 140 --boot order=scsi0
```

### Step 4 — Configure boot order

Proxmox web UI → VM 140 → Options → Boot Order:
- Enable `scsi0` (the imported HAOS disk)
- Disable all others

### Step 5 — Start the VM

```bash
qm start 140
```

Wait 2–3 minutes for first boot. HAOS will resize the disk to fill
the allocated storage automatically.

---

## Phase 2 — Initial Home Assistant Setup

### Step 1 — Access the onboarding UI

Navigate to `http://10.1.20.10:8123` from a browser on VLAN 10.

> Use HTTP directly during initial setup — Traefik is not yet
> configured. You'll access via `https://homeassistant.eirdom.homes`
> after the Traefik route is added in Phase 3.

### Step 2 — Complete the onboarding wizard

1. Create your Home Assistant account:
   - Name: Tyler
   - Username: `tyler`
   - Password: strong password → save to password manager
2. Set home location (used for sunrise/sunset automations)
3. Select your timezone: `America/Chicago`
4. Skip device discovery for now — add integrations after setup

### Step 3 — Configure a static IP

By default HAOS uses DHCP. Set a static IP to match the spec:

Settings → System → Network → IPv4:
- Address: `10.1.20.10/24`
- Gateway: `10.1.20.1`
- DNS: `10.1.10.10`

Apply and wait for reconnection.

---

## Phase 3 — Traefik Integration

Add a static route to `docker/traefik/dynamic/routers.yml`:

```yaml
    # -----------------------------------------------------------
    # Home Assistant
    # Internal only — public chain (HA handles own auth)
    # HAOS uses self-signed cert — skip TLS verification
    # -----------------------------------------------------------
    homeassistant:
      rule: "Host(`homeassistant.${ROOT_DOMAIN}`)"
      entryPoints:
        - websecure
      service: homeassistant-svc
      tls:
        certResolver: cloudflare
      middlewares:
        - chain-public

  # (add to services section)
    homeassistant-svc:
      loadBalancer:
        servers:
          - url: "https://10.1.20.10:8123"
        serversTransport: proxmox-transport
```

Add the DNS record on EIRDOM-DC-01:

```
homeassistant    A    10.1.50.10
```

After updating `routers.yml`, Traefik picks it up within seconds —
no restart needed. Verify at `https://homeassistant.eirdom.homes`.

---

## Phase 4 — Firewall Rules

HAOS on VLAN 20 needs two exceptions added to the UniFi firewall.

In UniFi Network → Firewall & Security → Firewall Rules,
add to the **IOT zone**:

### Rule 1 — HA DNS resolution

| Field | Value |
|-------|-------|
| Name | `ha_corp_dns` |
| Source | `obj_eirdom_haos` (IP object: 10.1.20.10) |
| Destination | `obj_eirdom_dc01` (10.1.10.10) |
| Protocol/Port | TCP/UDP 53 |
| Action | Allow |
| Position | Before `iot_corporate_deny` |

### Rule 2 — HA → Traefik (internal service access)

| Field | Value |
|-------|-------|
| Name | `ha_docker_traefik` |
| Source | `obj_eirdom_haos` (10.1.20.10) |
| Destination | `obj_traefik` (10.1.50.10) |
| Protocol/Port | TCP 443 |
| Action | Allow |
| Position | Before `iot_corporate_deny` |

> These rules are scoped to `obj_eirdom_haos` (10.1.20.10) only —
> not the entire VLAN 20 subnet. No other IoT device gains corporate
> access. Update `unifi/firewall/lan-rules.md` to document both rules.

---

## Phase 5 — Key Integrations

### UniFi Protect (Cameras)

The UniFi Protect integration works fully with Fabric — it uses the
Protect local API separately from Network, which still supports direct
access.

Settings → Integrations → Add → UniFi Protect:
- Host: `10.1.1.1` (UDM-Pro-Max)
- Username / Password: your Ubiquiti account credentials
- Verify SSL: off

This gives Home Assistant:
- Live camera feeds for all G5 Pro, G6 Instant 180, AI Theta, G5 Dome
- Motion events as HA triggers for automations
- Doorbell press events (front door G4 Doorbell Pro PoE)
- Smart detection events (person, vehicle, package)

### Mobile Companion App (iOS / Android)

Install **Home Assistant** from the App Store / Google Play.

Server URL: `https://homeassistant.eirdom.homes`

The companion app provides:
- Presence detection via GPS (replaces network-based device tracking)
- Push notifications from HA automations
- Device sensors (battery, location, activity, Wi-Fi SSID)
- Remote control of all HA entities

### Ntfy (Push Notifications)

Install the ntfy add-on via HACS or configure the RESTful notify
integration to publish to your ntfy server:

Settings → Integrations → Add → RESTful:
- URL: `https://ntfy.eirdom.homes`
- Method: POST

Or use the ntfy HACS integration for better native support.

This routes HA automation notifications through your self-hosted
ntfy server to phones — no cloud notification service needed.

### Jellyfin (Media Control)

Settings → Integrations → Add → Jellyfin:
- URL: `https://jellyfin.eirdom.homes`
- Username / Password: your Jellyfin AD credentials

Enables media player control entities, now-playing status on
dashboards, and media-aware automations (e.g. dim lights when
playback starts).

### Zigbee / Z-Wave (Smart Devices)

For USB Zigbee (e.g. Sonoff Zigbee 3.0, HUSBZB-1) or Z-Wave
dongles, pass through the USB device from Proxmox:

```bash
# On EIRDOM-PVE-01 — find the USB device ID
lsusb

# Add USB passthrough to VM 140
qm set 140 --usb0 host=VENDOR:PRODUCT
```

Then in HAOS → Settings → Integrations → Add → Zigbee Home
Automation (ZHA) or Z-Wave JS.

> Matter and Thread are also supported natively on recent HAOS
> versions — no USB dongle needed for Matter devices.

---

## Phase 6 — Useful Add-ons

Install via Settings → Add-ons → Add-on Store:

| Add-on | Purpose |
|--------|---------|
| **File Editor** | Edit HA config files from the browser |
| **Terminal & SSH** | SSH access to HAOS |
| **Node-RED** | Visual automation builder (more powerful than built-in) |
| **AppDaemon** | Python-based automations for complex logic |
| **Mosquitto Broker** | Local MQTT server for devices that need it |
| **Z-Wave JS** | Z-Wave device management (if using Z-Wave) |
| **Zigbee2MQTT** | Alternative to ZHA for Zigbee (broader device support) |

---

## Backup Strategy

HAOS has a built-in backup system. Configure automatic backups:

Settings → System → Backups → Automatic backups:
- Schedule: Daily
- Retention: 7 days
- Location: `/backup` (on the VM disk)

For off-VM backup, configure the **Samba Backup** add-on to copy
backup files to a NAS or to `${MEDIA_PATH}/homeassistant/backups/`
on EIRDOM-DOCKER-01 via SSH/SCP.

Also configure Proxmox VM backup:

Datacenter → Backup → Add:
- VM: `140` (EIRDOM-HAOS-01)
- Schedule: Weekly (Sunday 2:00 AM)
- Retention: Keep last 4

---

## Troubleshooting

### HA unreachable at homeassistant.eirdom.homes

1. Verify the VM is running: `qm status 140` on EIRDOM-PVE-01
2. Verify HA is accessible directly at `http://10.1.20.10:8123`
3. Check Traefik logs for routing errors:
   ```bash
   docker logs traefik --tail 20
   ```
4. Verify DNS record `homeassistant.eirdom.homes → 10.1.50.10`
   exists on EIRDOM-DC-01

### Smart devices not discovered

Verify the device is on VLAN 20 and HA can reach it:

```bash
# From HAOS terminal (SSH or web terminal add-on)
ping 10.1.20.XXX
```

If the device is on a different VLAN, add a firewall rule allowing
HA (`10.1.20.10`) to reach that VLAN's subnet.

### Protect integration fails

Verify HA can reach the UDM-Pro-Max on port 443:

```bash
# From HAOS terminal
curl -k https://10.1.1.1
```

If blocked, add a firewall rule: `ha_udm_protect` — source
`10.1.20.10` → destination `10.1.1.1` TCP 443, Allow, before
`iot_corporate_deny`.

---

## Related Documentation

- [`docs/services.md`](services.md) — EIRDOM-HAOS-01 service entry
- [`docs/deployment-guide.md`](deployment-guide.md) — Phase 17
- [`docs/decisions.md`](decisions.md) — ADR-041
- [`docs/cron.md`](cron.md) — Proxmox backup schedule
- [`unifi/firewall/lan-rules.md`](../unifi/firewall/lan-rules.md) — HA firewall rules
- [`docker/traefik/dynamic/routers.yml`](../docker/traefik/dynamic/routers.yml) — Traefik route
- [`docker/ntfy/README.md`](../docker/ntfy/README.md) — Ntfy push notifications