# Wazuh — Server Deployment Guide
> EIRDOM-WAZUH-01 · VM 120 · VLAN 60 · 10.1.60.10
> Wazuh Manager + Indexer + Dashboard (All-in-One)
> Phase 5 — Security Monitoring
> Last Updated: April 2026

---

## Overview

Wazuh provides unified XDR and SIEM capabilities across the entire
Eirdom infrastructure:

- **File integrity monitoring** — detects changes to critical files
  and directories on all endpoints
- **Vulnerability detection** — identifies unpatched CVEs on agents
- **Security configuration assessment** — benchmarks endpoints against
  CIS standards
- **Log analysis** — aggregates and correlates logs from agents, the
  UDM-Pro-Max, and Traefik
- **Active response** — automated blocking of known-bad IPs

This guide covers standing up the Wazuh server VM on Proxmox.
Agent deployment is covered separately in `docs/wazuh-agents.md`.

---

## VM Specification

| Field | Value |
|-------|-------|
| VM ID | 120 |
| Hostname | `EIRDOM-WAZUH-01` |
| FQDN | `EIRDOM-WAZUH-01.ad.eirdom.homes` |
| OS | Ubuntu 26.04 LTS Server |
| vCPU | 4 (minimum) — 8 recommended |
| RAM | 8 GB (minimum) — 16 GB recommended |
| Disk | 200 GB (stores 90 days of indexed alerts) |
| Network | VLAN 60 |
| IP | 10.1.10.10/24 (static) |
| Gateway | 10.1.60.1 |
| DNS | 10.1.10.10 (EIRDOM-DC-01) |
| Internal URL | `https://wazuh.eirdom.homes` |

> **RAM note:** Wazuh's OpenSearch Indexer is memory-hungry. 8 GB is
> workable for a home lab but the dashboard may feel sluggish when
> running complex queries. 16 GB provides a noticeably better
> experience. Proxmox makes it easy to add RAM later — start with 8
> and expand if needed.

---

## Phase 1 — Create the VM in Proxmox

### Step 1 — Download Ubuntu 24.04 ISO

In the Proxmox web UI (`https://proxmox.eirdom.homes`):

1. Navigate to **local (pve-01) → ISO Images**
2. Click **Download from URL**
3. URL: `https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso`
4. Click **Query URL** then **Download**

### Step 2 — Create the VM

**General:**
- VM ID: `120`
- Name: `EIRDOM-WAZUH-01`

**OS:**
- ISO Image: `ubuntu-24.04-live-server-amd64.iso`
- Guest OS Type: Linux, Version: 6.x - 2.6 Kernel

**System:**
- Machine: `q35`
- BIOS: `OVMF (UEFI)`
- Add EFI Disk: checked
- SCSI Controller: `VirtIO SCSI single`
- Qemu Agent: checked

**Disks:**
- Bus/Device: `SCSI 0`
- Storage: your VM storage pool
- Disk size: `200 GB`
- Cache: `Write back`
- Discard: checked (if using SSD/NVMe storage)

**CPU:**
- Sockets: `1`
- Cores: `4`
- Type: `host`

**Memory:**
- Memory: `8192 MB` (8 GB)
- Ballooning: disabled (Wazuh benefits from dedicated RAM)

**Network:**
- Bridge: `vmbr0`
- VLAN Tag: `60`
- Model: `VirtIO`
- Firewall: unchecked (handled by UDM-Pro-Max)

Click **Finish** (do not start yet).

### Step 3 — Start and install Ubuntu

1. Start the VM and open the console
2. Follow the Ubuntu Server installer:
   - Language: English
   - Keyboard: US
   - Network: configure the VirtIO NIC
     - IP: `10.1.60.10/24`
     - Gateway: `10.1.60.1`
     - DNS: `10.1.10.10`
     - Search domain: `ad.eirdom.homes`
   - Storage: use the full 200 GB disk, LVM layout
   - Profile:
     - Your name: `Wazuh Admin`
     - Server name: `EIRDOM-WAZUH-01`
     - Username: `wazuhadmin`
     - Password: generate strong password, store in password manager
   - SSH: Install OpenSSH server — **checked**
   - Featured snaps: none — skip all
3. Complete install and reboot

### Step 4 — Post-install configuration

SSH to the VM from EIRDOM-DOCKER-01 or your workstation:

```bash
ssh wazuhadmin@10.1.60.10
```

**Install QEMU guest agent:**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

**Set static hostname:**

```bash
sudo hostnamectl set-hostname EIRDOM-WAZUH-01
echo "10.1.60.10 EIRDOM-WAZUH-01 EIRDOM-WAZUH-01.ad.eirdom.homes" | \
  sudo tee -a /etc/hosts
```

**Join the AD domain (optional but recommended):**

```bash
sudo apt install -y sssd realmd adcli krb5-user samba-common-bin
sudo realm join ad.eirdom.homes -U Administrator
```

---

## Phase 2 — Install Wazuh (All-in-One)

The all-in-one installer deploys Wazuh Manager, OpenSearch Indexer,
and the Wazuh Dashboard on a single host. This is the recommended
approach for home lab scale.

### Step 1 — Download and run the installer

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

This takes 10–20 minutes. The installer will:

- Install and configure the Wazuh Indexer (OpenSearch)
- Install and configure the Wazuh Manager
- Install and configure the Wazuh Dashboard
- Generate self-signed TLS certificates for all components
- Start all services

### Step 2 — Save the admin credentials

At the end of the install, credentials are displayed:

```
Admin password: <generated>
Kibana user:    kibanaserver
Kibana pass:    <generated>
```

**Save these immediately to your password manager.** They will not
be displayed again. If lost, run:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

### Step 3 — Verify services are running

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

All three should show `active (running)`.

### Step 4 — Access the dashboard

Navigate to `https://10.1.60.10` in a browser. Accept the self-signed
certificate warning for now — Traefik will handle proper TLS.

Log in with:
- **Username:** `admin`
- **Password:** the admin password saved above

---

## Phase 3 — Integrate with Traefik

Traefik is already configured to reverse proxy the Wazuh dashboard
at `https://wazuh.eirdom.homes` via `docker/traefik/dynamic/routers.yml`.

The route uses `internal-transport` (standard TLS verification) but
Wazuh uses a self-signed certificate. Update the transport to skip
verification for Wazuh:

In `docker/traefik/dynamic/routers.yml`, the `wazuh-svc` entry
already exists pointing to `https://10.1.60.10:443`. Verify it uses
`internal-transport` — if Wazuh's self-signed cert causes issues,
temporarily switch it to `proxmox-transport` (which has
`insecureSkipVerify: true`) until you replace the cert.

**Add a DNS record** on EIRDOM-DC-01 if not already present:

```
wazuh.eirdom.homes    A    10.1.50.10
```

Test access at `https://wazuh.eirdom.homes` — you should see the
Wazuh dashboard through Traefik with a valid certificate.

> The dashboard is protected by `chain-admin` middleware — only
> accessible from VLAN 10 (Corporate) and requires Authentik SSO.

---

## Phase 4 — Configure UDM-Pro-Max Syslog

The UDM-Pro-Max forwards firewall deny logs, IDS/IPS alerts, and
DHCP events to Wazuh via syslog.

**In UniFi Network:**

Settings → System → Logging → Remote Logging:
- Enable remote logging: **On**
- Server IP: `10.1.60.10`
- Port: `514`
- Protocol: `UDP`

After saving, firewall events from the UDM will begin appearing in
Wazuh within a few minutes.

**Verify in Wazuh dashboard:**

Wazuh → Threat Intelligence → Events → filter by `agent.name: EIRDOM-WAZUH-01`

You should see syslog events arriving from `10.1.1.1`.

---

## Phase 5 — Configure Traefik Log Forwarding

Traefik access logs from EIRDOM-DOCKER-01 contain valuable data
about HTTP requests, errors, and potential attacks. Forward them to
Wazuh via the agent installed on EIRDOM-DOCKER-01.

After deploying the Wazuh agent on EIRDOM-DOCKER-01 (see
`docs/wazuh-agents.md`), add this to the agent's `ossec.conf`:

```xml
<!-- Traefik access log — JSON format -->
<localfile>
  <log_format>json</log_format>
  <location>/media/arr/config/traefik/logs/access.log</location>
</localfile>

<!-- Traefik error log -->
<localfile>
  <log_format>json</log_format>
  <location>/media/arr/config/traefik/logs/traefik.log</location>
</localfile>
```

Restart the agent after editing:

```bash
sudo systemctl restart wazuh-agent
```

---

## Phase 6 — Initial Configuration in Wazuh Dashboard

### Create agent groups

Before deploying agents, create the groups that will control their
configuration. See `docs/wazuh-agents.md` for the full group list.

**Wazuh Dashboard → Management → Groups → Add new group:**

- `windows-workstations`
- `windows-servers`
- `linux-servers`

### Configure alerts for your environment

Navigate to **Wazuh → Management → Rules** and note the default rule
levels. For a home lab, sensible initial thresholds:

| Alert Level | Action | Example |
|-------------|--------|---------|
| 3–6 | Log only | Informational events |
| 7–10 | Dashboard alert | Failed logins, config changes |
| 11–15 | Email alert (once configured) | Rootkits, active attacks |

### Set up email alerts (optional)

Edit `/var/ossec/etc/ossec.conf` on EIRDOM-WAZUH-01:

```xml
<global>
  <email_notification>yes</email_notification>
  <smtp_server>smtp.gmail.com</smtp_server>
  <email_from>wazuh@eirdom.homes</email_from>
  <email_to>tyler.eirdom@outlook.com</email_to>
  <email_maxperhour>12</email_maxperhour>
  <email_log_source>alerts.log</email_log_source>
</global>
```

Restart Wazuh Manager after editing:

```bash
sudo systemctl restart wazuh-manager
```

### Set up Ntfy push alerts (recommended)

This sends critical Wazuh alerts directly to your phone via Ntfy.
Requires Ntfy to be deployed and a service token generated.

Create a custom integration script on EIRDOM-WAZUH-01:

```bash
sudo mkdir -p /var/ossec/integrations
sudo tee /var/ossec/integrations/custom-ntfy << 'EOF'
#!/bin/bash
# Wazuh → Ntfy push notification integration
# Triggered by Wazuh for alerts at the configured level

ALERT_FILE="$1"
NTFY_URL="https://ntfy.eirdom.homes/eirdom-security"
NTFY_TOKEN="YOUR_NTFY_TOKEN_HERE"

# Extract alert details
LEVEL=$(python3 -c "import sys,json; d=json.load(open('$ALERT_FILE')); print(d.get('rule',{}).get('level',''))" 2>/dev/null)
DESC=$(python3 -c "import sys,json; d=json.load(open('$ALERT_FILE')); print(d.get('rule',{}).get('description','Unknown alert'))" 2>/dev/null)
AGENT=$(python3 -c "import sys,json; d=json.load(open('$ALERT_FILE')); print(d.get('agent',{}).get('name','Unknown'))" 2>/dev/null)

curl -s \
    -H "Authorization: Bearer $NTFY_TOKEN" \
    -H "Title: Wazuh Alert — Level $LEVEL" \
    -H "Priority: high" \
    -H "Tags: warning" \
    -d "Agent: $AGENT | $DESC" \
    "$NTFY_URL" > /dev/null 2>&1

exit 0
EOF
sudo chmod 750 /var/ossec/integrations/custom-ntfy
sudo chown root:wazuh /var/ossec/integrations/custom-ntfy
```

Replace `YOUR_NTFY_TOKEN_HERE` with the token generated via
`docker exec ntfy ntfy token add tyler` on EIRDOM-DOCKER-01.

Then register the integration in `/var/ossec/etc/ossec.conf`:

```xml
<integration>
  <name>custom-ntfy</name>
  <level>12</level>
  <alert_format>json</alert_format>
</integration>
```

Restart Wazuh Manager:

```bash
sudo systemctl restart wazuh-manager
```

Test by triggering a test alert:

```bash
# Generate a test level-12 alert
sudo /var/ossec/bin/ossec-logtest
```

You should receive a push notification on your phone within
seconds. Adjust `<level>` to control alert threshold —
12 catches serious events without excessive noise.

---

## Phase 7 — Proxmox Backup

Configure Wazuh VM backup in Proxmox web UI:

**Datacenter → Backup → Add:**
- Node: `pve-01`
- VM: `120` (EIRDOM-WAZUH-01)
- Schedule: Weekly (Sunday 1:00 AM)
- Mode: Snapshot
- Retention: Keep last 4

This is already in `docs/cron.md` as part of the Proxmox backup
schedule.

---

## Ongoing Maintenance

### Checking manager status

```bash
# Service status
sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard

# Agent connections
sudo /var/ossec/bin/agent_control -l

# Manager logs
sudo tail -f /var/ossec/logs/ossec.log
```

### Updating Wazuh

```bash
# Check current version
sudo /var/ossec/bin/wazuh-control info

# Update all components
sudo apt update
sudo apt install --only-upgrade wazuh-manager wazuh-indexer wazuh-dashboard
```

> Always check the Wazuh release notes before upgrading:
> `documentation.wazuh.com/current/release-notes/`

### Disk space management

Wazuh's OpenSearch Indexer retains indexed alerts. Monitor disk usage:

```bash
df -h /var/lib/wazuh-indexer
```

If disk fills up, adjust the index retention policy in the Wazuh
dashboard under **Indexer management → Index policies**.

---

## Troubleshooting

### Dashboard not loading

```bash
sudo systemctl status wazuh-dashboard
sudo journalctl -u wazuh-dashboard --since "10 minutes ago"
```

Most common cause: OpenSearch Indexer not fully started yet.
Give it 2–3 minutes after boot before the dashboard becomes
responsive.

### Agents not connecting

Wazuh agents communicate on **TCP/UDP 1514** (agent comms) and
**TCP 1515** (registration). Verify:

```bash
# Test from agent side
nc -zv 10.1.60.10 1514
nc -zv 10.1.60.10 1515
```

The `security_all_allow` firewall rule permits all traffic from VLAN
60 outbound, but check that the source VLAN can reach VLAN 60.

### High memory usage

OpenSearch is memory-intensive. If the VM is swapping:

```bash
free -h
```

Either increase VM RAM in Proxmox (recommended) or reduce the
OpenSearch heap size:

```bash
# Edit /etc/wazuh-indexer/jvm.options
# Change -Xms4g and -Xmx4g to -Xms2g and -Xmx2g
sudo systemctl restart wazuh-indexer
```

---

## Next Step — Deploy Agents

With the Wazuh server running, the next step is deploying agents to
all endpoints. See [`docs/wazuh-agents.md`](wazuh-agents.md) for
the complete per-endpoint deployment guide covering:

- GPO-based deployment to Windows workstations
- Manual deployment to EIRDOM-DC-01 and EIRDOM-SUB-01
- Agent deployment to EIRDOM-DOCKER-01 and EIRDOM-PVE-01
- Traefik log forwarding configuration
- Agent group assignment and verification

---

## Related Documentation

- [`docs/wazuh-agents.md`](wazuh-agents.md) — Agent deployment (next step)
- [`docs/services.md`](services.md) — Wazuh service entry (VM 120)
- [`docs/decisions.md`](decisions.md) — ADR-022
- [`docs/cron.md`](cron.md) — Proxmox backup schedule
- [`docs/network-diagram.md`](network-diagram.md) — Security monitoring data flow
- [`unifi/firewall/ids-ips.md`](../unifi/firewall/ids-ips.md) — UDM syslog config
- [`unifi/firewall/lan-rules.md`](../unifi/firewall/lan-rules.md) — SECURITY zone rules