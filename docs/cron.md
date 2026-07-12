# Eirdom — Scheduled Tasks
> Cron jobs and automated schedules across the infrastructure
> Last Updated: April 2026

---

## Overview

This document lists every scheduled task running across the Eirdom
infrastructure. Keep it updated when adding new automation so there
is one place to audit what runs and when.

---

## EIRDOM-DOCKER-01 (10.1.50.10)

### Setup

All cron jobs on EIRDOM-DOCKER-01 run as the `dockeradmin` user
unless noted. Edit with:

```bash
crontab -e
```

### Current Cron Jobs

```cron
# =============================================================
# Eirdom — EIRDOM-DOCKER-01 Cron Jobs
# dockeradmin user
# =============================================================

# -----------------------------------------------------------
# Daily backup — configs, DB dumps, repo snapshot
# Runs at 2:00 AM every day
# Logs to scripts/logs/backup_<timestamp>.log
# -----------------------------------------------------------
0 2 * * * cd /path/to/eirdom && sudo bash scripts/backup.sh

# -----------------------------------------------------------
# Weekly Docker image updates
# Runs at 3:00 AM every Sunday
# Only updates images — does not handle DB migrations
# Review update.sh output before running on production
# Logs to scripts/logs/update_<timestamp>.log
# -----------------------------------------------------------
# 0 3 * * 0 cd /path/to/eirdom && sudo bash scripts/update.sh
# NOTE: Commented out — run manually to review changelogs first
# Uncomment once you are comfortable with the update cadence

# -----------------------------------------------------------
# NetOps Toolkit — weekly network documentation snapshot
# Runs at 2:00 AM every Sunday
# Polls UDM-Pro-Max API, generates AI documentation via Claude
# Logs to tools/netops-toolkit/output/logs/
# Requires: venv set up, config/config.yaml filled in,
#           ANTHROPIC_API_KEY and UNIFI_PASSWORD in environment
# -----------------------------------------------------------
0 2 * * 0 cd /path/to/eirdom/tools/netops-toolkit \
  && /path/to/eirdom/tools/netops-toolkit/venv/bin/python \
     scripts/collect_and_document.py \
  >> /var/log/eirdom-netops.log 2>&1
```

### Replacing `/path/to/eirdom`

After cloning the repo on the server, replace all instances of
`/path/to/eirdom` with the actual repo path. Typically:

```bash
REPO_PATH=$(pwd)  # run from inside the cloned repo
sed -i "s|/path/to/eirdom|$REPO_PATH|g" /path/to/crontab_file
```

### Installing the Cron Jobs

```bash
# View current crontab
crontab -l

# Edit and add the jobs above
crontab -e

# Verify
crontab -l
```

---

## Recyclarr (Docker container — internal schedule)

Recyclarr runs on its own internal schedule managed by the container.
No host cron job needed.

| Schedule | Task | Configuration |
|----------|------|---------------|
| Daily (default) | Sync TRaSH Guides quality profiles to Radarr × 2, Sonarr × 2 | `${DOCKER_DATA_PATH}/recyclarr/recyclarr.yml` |

To change the sync interval, set `RECYCLARR_CRON_SYNC` in the
arr-stack `.env`. Example for weekly sync on Sunday at 4 AM:

```bash
RECYCLARR_CRON_SYNC=0 4 * * 0
```

To trigger a manual sync at any time:

```bash
docker exec recyclarr recyclarr sync
```

---

## NetBox (Docker container — internal schedule)

The `netbox_unifi_sync` plugin runs sync jobs via NetBox's built-in
scheduled job system. No host cron job needed.

| Schedule | Task | Configure In |
|----------|------|-------------|
| Every 60 minutes (recommended) | UniFi → NetBox device/VLAN sync | NetBox UI → Plugins → UniFi Sync → Schedule |

To trigger a manual sync:

```bash
docker exec netbox /opt/netbox/venv/bin/python \
  /opt/netbox/netbox/manage.py netbox_unifi_sync_run
```

---

## EIRDOM-DC-01 (10.1.10.10) — Windows Task Scheduler

### Wazuh Agent Health Check

Windows Task Scheduler (not cron) manages Windows-side automation.
No scheduled tasks are currently defined for EIRDOM-DC-01 beyond
the defaults created during AD DS and Wazuh agent deployment.

Future tasks to consider:

- AD replication health report
- DHCP scope utilization alert
- Certificate expiry check for EIRDOM-SUB-01

---

## Proxmox (EIRDOM-PVE-01 — 10.1.10.5)

Proxmox backup jobs are configured in the Proxmox web UI, not cron.

| Schedule | Task | Retention | Destination |
|----------|------|-----------|-------------|
| Daily 1:00 AM | Backup EIRDOM-DC-01 (VM 100) | 7 daily | NAS or local storage |
| Weekly Sunday 1:00 AM | Backup EIRDOM-SUB-01 (VM 102) | 4 weekly | NAS or local storage |
| Weekly Sunday 1:30 AM | Backup EIRDOM-WAZUH-01 (VM 120) | 4 weekly | NAS or local storage |
| Weekly Sunday 1:30 AM | Backup EIRDOM-SONION-01 (VM 130) | 4 weekly | NAS or local storage |
| Weekly Sunday 2:00 AM | Backup EIRDOM-HAOS-01 (VM 140) | 4 weekly | NAS or local storage |

> VM 101 (Offline Root CA) is air-gapped and powered off — it is
> not included in automated Proxmox backup schedules. Back it up
> manually to encrypted USB after any change. See `docs/decisions.md`
> ADR-004 for details.

Configure in Proxmox web UI: **Datacenter → Backup → Add**

---

## Annual Manual Tasks

These are not automated — add calendar reminders:

| When | Task | Reference |
|------|------|-----------|
| Annually (April) | Power on Offline Root CA, publish new CRL, shut down | `docs/decisions.md` ADR-004 |
| Annually | Renew `eirdom.homes` domain registration in Cloudflare | Cloudflare dashboard |
| Annually | Full infrastructure documentation review | This repo |
| Annually | Review and rotate all service account passwords | `docs/ad-groups.md` |
| Annually | Review Wazuh SCA compliance and remediate drift | Wazuh dashboard |

---

## Log Locations

| Task | Log Location |
|------|-------------|
| `backup.sh` | `scripts/logs/backup_<timestamp>.log` |
| `update.sh` | `scripts/logs/update_<timestamp>.log` |
| NetOps toolkit | `tools/netops-toolkit/output/logs/` + `/var/log/eirdom-netops.log` |
| Recyclarr | `docker logs recyclarr` |
| NetBox UniFi sync | `docker logs netbox-worker` |
| Traefik access | `${DOCKER_DATA_PATH}/traefik/logs/access.log` |
| Traefik errors | `${DOCKER_DATA_PATH}/traefik/logs/traefik.log` |

---

## Related Documentation

- [`scripts/backup.sh`](../scripts/backup.sh) — Backup script
- [`scripts/update.sh`](../scripts/update.sh) — Update script
- [`tools/netops-toolkit/README.md`](../tools/netops-toolkit/README.md) — NetOps toolkit
- [`docker/arr-stack/README.md`](../docker/arr-stack/README.md) — Recyclarr schedule
- [`docker/netbox/README.md`](../docker/netbox/README.md) — NetBox sync schedule