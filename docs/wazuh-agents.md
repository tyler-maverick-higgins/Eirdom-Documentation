# Wazuh Agent Deployment
> Phase 5 — Security Monitoring
> EIRDOM-WAZUH-01 (10.1.60.10) · Wazuh 4.x
> Last Updated: April 2026

---

## Overview

Wazuh agents provide endpoint visibility — file integrity monitoring,
vulnerability detection, security configuration assessment, log
forwarding, and active response. This document covers deploying agents
to every endpoint in the Eirdom infrastructure.

**Wazuh Manager must be running before deploying any agents.**
Verify at `https://wazuh.eirdom.homes` before proceeding.

---

## Agent Groups

Create these groups in Wazuh before deploying agents. Groups control
which configuration and rules apply to each class of endpoint.

In Wazuh dashboard → **Management → Groups → Add new group:**

| Group Name | Targets | Key Config |
|------------|---------|-----------|
| `windows-workstations` | Family PCs and laptops | FIM on user dirs, Sysmon logs, CIS Windows benchmark |
| `windows-servers` | EIRDOM-DC-01, EIRDOM-SUB-01 | FIM on AD dirs, CIS Server benchmark, audit log forwarding |
| `linux-servers` | EIRDOM-DOCKER-01, EIRDOM-PVE-01 | FIM on /etc and /opt, CIS Debian/Ubuntu benchmark, Docker monitoring |

---

## Windows Agent Deployment

### Method 1 — GPO (Recommended for domain-joined machines)

This deploys the agent automatically to all domain-joined Windows
machines without manual intervention.

**On EIRDOM-DC-01:**

1. Download the Wazuh Windows agent MSI:
   ```
   https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi
   ```

2. Copy the MSI to the NETLOGON share:
   ```
   copy wazuh-agent-4.x.x-1.msi \\EIRDOM-DC-01\NETLOGON\wazuh-agent.msi
   ```

3. Open **Group Policy Management** on EIRDOM-DC-01

4. Edit the **Eirdom-Wazuh-Agent** GPO (linked to Computers OU):

5. Navigate to:
   `Computer Configuration → Policies → Windows Settings → Scripts → Startup`

6. Add a new startup script with the following command:
   ```batch
   msiexec /i \\EIRDOM-DC-01\NETLOGON\wazuh-agent.msi /q ^
     WAZUH_MANAGER="10.1.60.10" ^
     WAZUH_REGISTRATION_SERVER="10.1.60.10" ^
     WAZUH_AGENT_GROUP="windows-workstations"
   ```

7. For servers, use a separate GPO linked to the Servers OU with:
   ```batch
   WAZUH_AGENT_GROUP="windows-servers"
   ```

8. Force GPO update to test:
   ```
   gpupdate /force
   ```

### Method 2 — Manual install (single machine)

Run on the target Windows machine as Administrator:

```powershell
# Download the installer
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi" `
  -OutFile "wazuh-agent.msi"

# Install silently
msiexec /i wazuh-agent.msi /q `
  WAZUH_MANAGER="10.1.60.10" `
  WAZUH_REGISTRATION_SERVER="10.1.60.10" `
  WAZUH_AGENT_GROUP="windows-workstations"

# Start the service
NET START WazuhSvc
```

---

## Linux Agent Deployment

### Ubuntu / Debian (EIRDOM-DOCKER-01)

```bash
# Add Wazuh repository
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
  gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
  --import && \
  chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" | \
  tee /etc/apt/sources.list.d/wazuh.list

# Install agent
apt-get update
WAZUH_MANAGER="10.1.60.10" apt-get install -y wazuh-agent

# Enable and start
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

### Proxmox Host (EIRDOM-PVE-01)

Proxmox runs Debian under the hood. Use the same steps as Ubuntu
above — the repository works for Debian as well.

```bash
# Same repo setup as above, then:
WAZUH_MANAGER="10.1.60.10" apt-get install -y wazuh-agent
systemctl enable --now wazuh-agent
```

---

## Per-Endpoint Deployment Checklist

### EIRDOM-DC-01 (Windows Server 2025)

```
Group: windows-servers
Special config:
  - FIM on: C:\Windows\SYSVOL, C:\Windows\NTDS
  - SCA: CIS Windows Server 2022 benchmark
  - Forward: Windows Security event logs (4624, 4625, 4648, 4768, 4769)
  - Forward: DNS debug logs (if enabled)
```

```powershell
msiexec /i wazuh-agent.msi /q `
  WAZUH_MANAGER="10.1.60.10" `
  WAZUH_REGISTRATION_SERVER="10.1.60.10" `
  WAZUH_AGENT_GROUP="windows-servers"
NET START WazuhSvc
```

---

### EIRDOM-SUB-01 (Windows Server 2025)

```
Group: windows-servers
Special config:
  - FIM on: C:\Windows\System32\CertSrv\CertEnroll
  - Monitor: certificate issuance events (Event 4870, 4886, 4887)
  - SCA: CIS Windows Server 2022 benchmark
```

Same install command as EIRDOM-DC-01.

---

### EIRDOM-DOCKER-01 (Ubuntu 24.04)

```
Group: linux-servers
Special config:
  - FIM on: /etc, /opt/eirdom (or repo path), /media/arr/config
  - SCA: CIS Ubuntu 24.04 benchmark
  - Docker monitoring: forward Docker daemon logs
  - Forward: Traefik access/error logs from ${DOCKER_DATA_PATH}/traefik/logs/
```

After installing the agent, configure Traefik log forwarding by
adding to `/var/ossec/etc/ossec.conf`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/media/arr/config/traefik/logs/access.log</location>
</localfile>

<localfile>
  <log_format>json</log_format>
  <location>/media/arr/config/traefik/logs/traefik.log</location>
</localfile>
```

Then restart the agent:

```bash
systemctl restart wazuh-agent
```

---

### EIRDOM-PVE-01 (Proxmox / Debian)

```
Group: linux-servers
Special config:
  - FIM on: /etc/pve, /etc/network, /var/lib/pve-cluster
  - Monitor: VM start/stop events from Proxmox task log
  - SCA: CIS Debian benchmark
```

---

### Family Workstations (Windows 10/11)

```
Group: windows-workstations
Special config:
  - FIM on: C:\Users\<username>\Documents, C:\Users\<username>\Desktop
  - SCA: CIS Windows 10/11 benchmark
  - Sysmon integration (if Sysmon deployed via GPO)
  - Monitor: USB device insertions
```

Deployed automatically via **Eirdom-Wazuh-Agent** GPO.

---

## Verifying Agent Registration

After deploying, verify agents appear in Wazuh dashboard:

**Wazuh → Agents → (list)**

Each agent should show:
- Status: **Active**
- Last keep alive: within the last few minutes
- Group: correct group name

If an agent shows **Disconnected:**

```bash
# On the agent (Linux)
systemctl status wazuh-agent
journalctl -u wazuh-agent --since "10 minutes ago"

# Check manager connectivity
nc -zv 10.1.60.10 1514
nc -zv 10.1.60.10 1515
```

Wazuh agents communicate on:
- **TCP/UDP 1514** — agent communication
- **TCP 1515** — agent registration

Both ports must be reachable from the agent to EIRDOM-WAZUH-01
(10.1.60.10). The `security_all_allow` firewall rule in
`unifi/firewall/lan-rules.md` permits all traffic from VLAN 60 to
all VLANs — but also ensure VLAN 10, 50 etc. can reach VLAN 60 on
these ports (check the Security → All Zones Allow rule covers return
traffic).

---

## UDM-Pro-Max Syslog Integration

The UDM-Pro-Max forwards syslog to Wazuh for firewall deny logs,
IDS/IPS alerts, and DHCP events.

**Configure in UniFi Network:**
Settings → System → Syslog → Enable → Server: `10.1.60.10` Port: `514`

This is already documented in `unifi/firewall/ids-ips.md` but
repeated here for completeness — it is part of the security
monitoring setup.

---

## Post-Deployment Tuning

After all agents are connected and reporting for 1–2 weeks:

1. **Suppress false positives** — review alerts in Wazuh dashboard
   and add local rules to suppress known-good events
2. **SCA baseline** — run Security Configuration Assessment on each
   group and work through the findings
3. **Active Response** — consider enabling automatic blocking of IPs
   that trigger repeated authentication failures

---

## Related Documentation

- [`docs/decisions.md`](decisions.md) — ADR-022
- [`docs/services.md`](services.md) — Wazuh service entry
- [`docs/network-diagram.md`](network-diagram.md) — Security monitoring data flow
- [`unifi/firewall/ids-ips.md`](../unifi/firewall/ids-ips.md) — IDS/IPS config
- [`unifi/firewall/lan-rules.md`](../unifi/firewall/lan-rules.md) — Security zone firewall rules