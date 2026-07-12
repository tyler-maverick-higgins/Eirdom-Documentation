# Eirdom Offsite Backup Architecture

**3-2-1 Backup Strategy with Cloudflare R2 + rclone**

*Supplement to Eirdom Infrastructure Guide v3 — April 2026*

---

## Overview

The Eirdom infrastructure currently backs up all VMs to local storage via Proxmox vzdump and all Docker service configs via the `backup.sh` script to NAS. This satisfies two legs of the 3-2-1 backup rule: three copies on two different media. What is missing is the offsite copy.

This document defines a low-cost offsite backup architecture that layers on top of the existing local backup strategy. It uses Cloudflare R2 as the offsite storage target and rclone as the transfer tool. R2 was chosen because you already have a Cloudflare account for the Eirdom Tunnel and DNS, and it offers a permanent free tier with zero egress fees.

### 3-2-1 Backup Rule

| Rule | Current State | After This Guide |
|------|---------------|------------------|
| 3 copies of data | VM backups on Proxmox local + NAS | Local + NAS + Cloudflare R2 |
| 2 different media types | NVMe SSD + NAS HDD | NVMe SSD + NAS HDD + Cloud Object Storage |
| 1 offsite copy | Not implemented | Cloudflare R2 (encrypted with rclone crypt) |

> **IMPORTANT:** This architecture does NOT replace Proxmox Backup Server (PBS). PBS provides deduplication, incremental backups, and fast restores for day-to-day operations. This offsite layer is strictly for disaster recovery: if the house burns down, you can rebuild from R2.

---

## Architecture

### What Gets Backed Up Offsite

Not everything needs offsite backup. Media libraries are replaceable. Infrastructure configuration and identity data are not. The offsite strategy focuses exclusively on critical, irreplaceable data.

| Tier | Data | Source | Offsite Frequency | Why |
|------|------|--------|-------------------|-----|
| Critical | EIRDOM-DC-01 (AD/DNS/DHCP/NPS) | vzdump `.vma.zst` | Daily | Identity and auth for entire network |
| Critical | EIRDOM-SUB-01 (Issuing CA) | vzdump `.vma.zst` | Weekly | All internal certificates depend on this |
| Critical | WordPress DB + files | `backup.sh` SQL dump + tar | Daily | Family hub content is irreplaceable |
| Critical | Docker Compose + configs | Private Git repo + tar | On change | Entire service stack definition |
| Important | Arr stack configs | `backup.sh` tar archives | Weekly | Hours of custom configuration |
| Important | Traefik config + certs | `backup.sh` tar archives | Weekly | Routing rules + internal PKI cert |
| Important | Wazuh config (not data) | vzdump `.vma.zst` | Weekly | Custom rules and decoders |
| Skip | Media libraries (movies, TV, music) | N/A | Never | Re-downloadable; too large for cloud |
| Skip | Security Onion full data | N/A | Never | Forensic data; huge; rebuild from scratch |

### Data Flow

**Proxmox VMs:** Proxmox vzdump runs on schedule (daily/weekly per VM) and writes compressed `.vma.zst` archives to local storage. A cron job on EIRDOM-PVE-01 runs rclone to sync these archives to an encrypted R2 bucket. Only the VMs listed above are synced offsite.

**Docker configs:** The existing `backup.sh` script on EIRDOM-DOCKER-01 creates timestamped tar archives of WordPress, Arr configs, and Traefik. A separate cron job runs rclone to sync the latest backup set to the same encrypted R2 bucket.

**Git repo:** All Docker Compose files and configs are already in a private GitHub repo. GitHub itself serves as a second offsite copy for this data. No additional R2 sync is needed for the repo.

The full data flow:

| Step | Source | Action | Destination |
|------|--------|--------|-------------|
| 1 | EIRDOM-PVE-01 | vzdump creates `.vma.zst` backup | Local Proxmox storage |
| 2 | EIRDOM-PVE-01 | rsync or NFS copy to NAS | NAS (LAN copy) |
| 3 | EIRDOM-PVE-01 | rclone sync (encrypted) to R2 | Cloudflare R2 bucket |
| 4 | EIRDOM-DOCKER-01 | `backup.sh` creates tar archives | NAS (LAN copy) |
| 5 | EIRDOM-DOCKER-01 | rclone sync (encrypted) to R2 | Cloudflare R2 bucket |

---

## Cloudflare R2 Setup

### Free Tier Limits

Cloudflare R2 offers a permanent free tier (no credit card required, no time limit) with the following monthly allowances:

| Resource | Free Allowance | Overage Cost |
|----------|---------------|--------------|
| Storage | 10 GB | $0.015 / GB-month |
| Class A operations (writes, lists) | 1 million | $4.50 / million |
| Class B operations (reads) | 10 million | $0.36 / million |
| Egress (data transfer out) | Unlimited | Free forever |
| Delete operations | Unlimited | Free forever |

> **KEY BENEFIT:** The free 10 GB is measured as average storage over the billing period, not a transfer cap. Zero egress fees means restoring backups costs nothing. This is the key advantage over Backblaze B2 or AWS S3 for disaster recovery.

### Storage Estimate

With rclone crypt encryption adding roughly 5-10% overhead:

| Backup Target | Compressed Size (est.) | Frequency | Retention | R2 Storage (est.) |
|---------------|----------------------|-----------|-----------|-------------------|
| EIRDOM-DC-01 | ~3-5 GB | Daily | 7 days | ~25-35 GB |
| EIRDOM-SUB-01 | ~2-3 GB | Weekly | 4 weeks | ~8-12 GB |
| WordPress (DB + files) | ~500 MB | Daily | 7 days | ~3.5 GB |
| Arr stack configs | ~200 MB | Weekly | 4 weeks | ~800 MB |
| Traefik config + certs | ~50 MB | Weekly | 4 weeks | ~200 MB |
| Wazuh VM | ~4-6 GB | Weekly | 4 weeks | ~16-24 GB |
| **TOTAL (estimated)** | | | | **~55-75 GB** |

> **COST NOTE:** At 55-75 GB you will exceed the 10 GB free tier and pay roughly $0.70-$1.00/month for the overage. To stay closer to free, reduce retention to 3 days for daily backups or exclude the Wazuh VM (it can be rebuilt from scratch). Even at full retention, this is far cheaper than any VPS or cloud PBS solution.

### Create the R2 Bucket

Log into the Cloudflare dashboard with your Eirdom account. Navigate to **R2 Object Storage** in the left sidebar.

Click **Create bucket**. Name it **eirdom-backups**. Select the region closest to you (or Auto for automatic). Leave all other settings at defaults.

### Create an API Token

In the R2 section, click **Manage R2 API Tokens**. Create a new token with these settings:

| Setting | Value |
|---------|-------|
| Token name | `eirdom-backup-rclone` |
| Permissions | Object Read & Write |
| Bucket scope | `eirdom-backups` (specific bucket only) |
| Client IP filtering | Your public IP (if static) or leave open |
| TTL | No expiration (or set annual renewal reminder) |

Save the **Access Key ID** and **Secret Access Key** in your password manager immediately. You will also need the **S3 endpoint URL** shown on the R2 settings page (format: `https://<account-id>.r2.cloudflarestorage.com`).

---

## rclone Configuration

### Install rclone on EIRDOM-PVE-01

```bash
apt update && apt install -y rclone
```

### Configure the R2 Remote

Run `rclone config` and create a new remote:

```
n) New remote → name: r2
Storage: s3
Provider: Cloudflare
access_key_id: <your R2 Access Key ID>
secret_access_key: <your R2 Secret Access Key>
endpoint: https://<account-id>.r2.cloudflarestorage.com
acl: private
```

### Add Encryption Layer (rclone crypt)

Never store backups unencrypted in the cloud. Create a crypt remote that wraps the R2 remote:

```
n) New remote → name: r2-crypt
Storage: crypt
remote: r2:eirdom-backups
filename_encryption: standard
directory_name_encryption: true
password: <generate strong password, save in password manager>
```

> **CRITICAL WARNING:** The rclone crypt password and salt are the ONLY way to decrypt your offsite backups. If you lose them, the data is permanently unrecoverable. Store the `rclone.conf` file (which contains these secrets) in your password manager alongside the R2 API token. Also export a copy to the same encrypted USB drives where you keep the Root CA backup.

### Verify the Connection

Test the encrypted remote works end-to-end:

```bash
echo 'test' > /tmp/rclone-test.txt
rclone copy /tmp/rclone-test.txt r2-crypt:/test/
rclone ls r2-crypt:/test/           # Should show rclone-test.txt
rclone ls r2:eirdom-backups         # Should show encrypted filename
rclone delete r2-crypt:/test/       # Clean up
```

Repeat this same installation and configuration on EIRDOM-DOCKER-01. You can copy the `rclone.conf` file between hosts:

```bash
scp /root/.config/rclone/rclone.conf root@10.1.50.10:/root/.config/rclone/
```

---

## Proxmox Offsite Sync Script

This script runs on EIRDOM-PVE-01 after vzdump completes. It syncs only the specified VM backups to R2, with configurable retention.

### Script: `/root/scripts/offsite-sync.sh`

```bash
#!/bin/bash
# ============================================================
# offsite-sync.sh - Sync Proxmox VM backups to Cloudflare R2
# Runs on EIRDOM-PVE-01 via cron after vzdump completes
# ============================================================
set -euo pipefail

# --- Configuration ---
RCLONE_REMOTE="r2-crypt"
RCLONE_BUCKET_PATH="proxmox"
BACKUP_DIR="/var/lib/vz/dump"
LOG_FILE="/var/log/offsite-sync.log"

# VM IDs to sync offsite (critical VMs only)
DAILY_VMS="100"              # EIRDOM-DC-01
WEEKLY_VMS="102 120"         # EIRDOM-SUB-01, EIRDOM-WAZUH-01

# Retention: number of backups to keep in R2 per VM
DAILY_RETENTION=7
WEEKLY_RETENTION=4

# --- Functions ---
log() { echo "$(date '+%Y-%m-%d %H:%M:%S') $1" | tee -a "$LOG_FILE"; }

sync_vm() {
    local vmid=$1
    local retention=$2
    local remote_path="${RCLONE_REMOTE}:/${RCLONE_BUCKET_PATH}/vm-${vmid}"

    # Find the latest backup for this VM
    local latest=$(ls -t "${BACKUP_DIR}"/vzdump-qemu-${vmid}-*.vma.zst 2>/dev/null | head -1)
    if [ -z "$latest" ]; then
        log "[WARN] No backup found for VM ${vmid}"
        return 1
    fi

    local filename=$(basename "$latest")
    local logfile="${latest%.vma.zst}.log"

    # Check if this backup is already in R2
    if rclone ls "${remote_path}/${filename}" 2>/dev/null | grep -q "${filename}"; then
        log "[SKIP] VM ${vmid}: ${filename} already in R2"
        return 0
    fi

    # Upload with chunking for large files
    log "[SYNC] VM ${vmid}: uploading ${filename}..."
    rclone copy "$latest" "$remote_path/" \
        --s3-chunk-size 100M \
        --s3-upload-concurrency 4 \
        --transfers 1 \
        --log-file "$LOG_FILE" \
        --log-level INFO

    # Also upload the backup log
    [ -f "$logfile" ] && rclone copy "$logfile" "$remote_path/"

    log "[OK]   VM ${vmid}: ${filename} uploaded"

    # Prune old backups in R2
    local count=$(rclone lsf "$remote_path/" --include "vzdump-qemu-${vmid}-*.vma.zst" | wc -l)
    if [ "$count" -gt "$retention" ]; then
        local to_delete=$((count - retention))
        log "[PRUNE] VM ${vmid}: removing ${to_delete} old backup(s)"
        rclone lsf "$remote_path/" --include "vzdump-qemu-${vmid}-*.vma.zst" | \
            sort | head -n "$to_delete" | while read f; do
            rclone delete "${remote_path}/${f}"
            log "[DEL]  VM ${vmid}: deleted ${f}"
        done
    fi
}

# --- Main ---
log "========== Offsite sync started =========="

DOW=$(date +%u)  # 1=Monday, 7=Sunday

# Daily VMs
for vmid in $DAILY_VMS; do
    sync_vm "$vmid" "$DAILY_RETENTION" || true
done

# Weekly VMs (run on Sundays only)
if [ "$DOW" -eq 7 ]; then
    for vmid in $WEEKLY_VMS; do
        sync_vm "$vmid" "$WEEKLY_RETENTION" || true
    done
fi

log "========== Offsite sync complete =========="
```

### Cron Schedule (EIRDOM-PVE-01)

Schedule the script to run 2 hours after your vzdump backup window. If vzdump runs at midnight:

```cron
# /etc/cron.d/offsite-sync
0 2 * * * root /root/scripts/offsite-sync.sh
```

---

## Docker Host Offsite Sync

This script runs on EIRDOM-DOCKER-01 after `backup.sh` completes. It syncs the latest Docker backup set to R2.

### Script: `/root/scripts/offsite-sync-docker.sh`

```bash
#!/bin/bash
# ============================================================
# offsite-sync-docker.sh - Sync Docker backups to Cloudflare R2
# Runs on EIRDOM-DOCKER-01 via cron after backup.sh
# ============================================================
set -euo pipefail

RCLONE_REMOTE="r2-crypt"
RCLONE_BUCKET_PATH="docker"
BACKUP_BASE="/path/to/nas/backups"   # Adjust to your NAS mount
LOG_FILE="/var/log/offsite-sync-docker.log"
RETENTION_DAYS=7

log() { echo "$(date '+%Y-%m-%d %H:%M:%S') $1" | tee -a "$LOG_FILE"; }

log "========== Docker offsite sync started =========="

# Find today's backup directory
TODAY=$(ls -td "${BACKUP_BASE}"/backup_* 2>/dev/null | head -1)
if [ -z "$TODAY" ]; then
    log "[ERROR] No backup directory found"
    exit 1
fi

DATESTAMP=$(basename "$TODAY")
REMOTE="${RCLONE_REMOTE}:/${RCLONE_BUCKET_PATH}/${DATESTAMP}"

# Sync the backup directory
log "[SYNC] Uploading ${DATESTAMP}..."
rclone copy "$TODAY" "$REMOTE/" \
    --s3-chunk-size 100M \
    --transfers 2 \
    --log-file "$LOG_FILE" \
    --log-level INFO

log "[OK] ${DATESTAMP} uploaded"

# Prune old backups in R2
rclone lsf "${RCLONE_REMOTE}:/${RCLONE_BUCKET_PATH}/" --dirs-only | sort | \
    head -n -${RETENTION_DAYS} | while read dir; do
    log "[PRUNE] Deleting ${dir}"
    rclone purge "${RCLONE_REMOTE}:/${RCLONE_BUCKET_PATH}/${dir}"
done

log "========== Docker offsite sync complete =========="
```

### Cron Schedule (EIRDOM-DOCKER-01)

Schedule to run 1 hour after `backup.sh`. If `backup.sh` runs at 1:00 AM:

```cron
# /etc/cron.d/offsite-sync-docker
0 2 * * * root /root/scripts/offsite-sync-docker.sh
```

---

## Restore Procedures

Offsite backups are useless if you cannot restore from them. Test these procedures quarterly.

### Restore a Proxmox VM from R2

On a fresh or rebuilt Proxmox host with rclone configured:

```bash
# 1. List available backups
rclone lsf r2-crypt:/proxmox/vm-100/

# 2. Download the backup you want to restore
rclone copy r2-crypt:/proxmox/vm-100/vzdump-qemu-100-2026_04_14-00_00_01.vma.zst \
    /var/lib/vz/dump/

# 3. Restore using qmrestore
qmrestore /var/lib/vz/dump/vzdump-qemu-100-2026_04_14-00_00_01.vma.zst 100
```

### Restore Docker Configs from R2

On a fresh Docker host with rclone configured:

```bash
# 1. List available backup sets
rclone lsf r2-crypt:/docker/ --dirs-only

# 2. Download the backup set
rclone copy r2-crypt:/docker/backup_20260414_010000/ /tmp/restore/

# 3. Extract and restore
cd /opt/eirdom
tar xzf /tmp/restore/wordpress_db_*.tar.gz
tar xzf /tmp/restore/wordpress_files_*.tar.gz
tar xzf /tmp/restore/arr_configs_*.tar.gz
# ... then docker compose up -d
```

> **REMINDER:** You need `rclone.conf` (with the crypt password) to decrypt anything from R2. Without it, the data is permanently lost. This is why storing `rclone.conf` in your password manager AND on the encrypted USB drives with the Root CA backup is essential.

---

## Monitoring and Alerting

### Log Monitoring

Both offsite sync scripts write to log files. Forward these to Wazuh for centralized monitoring by adding the log paths to the Wazuh agent configuration on each host:

```xml
<!-- /var/ossec/etc/ossec.conf on EIRDOM-PVE-01 -->
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/offsite-sync.log</location>
</localfile>
```

Create a custom Wazuh rule to alert on sync failures (any line containing `[ERROR]` or `[WARN]`).

### Cloudflare R2 Dashboard

Check the R2 metrics page in the Cloudflare dashboard weekly to verify storage is growing as expected. A sudden drop to zero indicates a misconfigured prune job. Add this to your weekly maintenance checklist.

### Quarterly Restore Test

Every quarter, pull a random VM backup from R2 and restore it to a temporary VM on Proxmox. Verify the VM boots and services start correctly. This validates the entire chain: vzdump, rclone encryption, R2 storage, rclone decryption, and qmrestore. Delete the temporary VM after testing. Add this to your existing quarterly maintenance schedule alongside the AD disaster recovery test.

---

## Updated Maintenance Schedule

Add the following items to the existing Eirdom maintenance schedule:

| Frequency | Task | Service |
|-----------|------|---------|
| Daily | Verify offsite sync logs have no errors | R2 / rclone |
| Weekly | Check R2 storage usage in Cloudflare dashboard | Cloudflare R2 |
| Monthly | Verify `rclone.conf` is current in password manager | rclone |
| Quarterly | Restore test: pull random VM from R2, boot on Proxmox | R2 / Proxmox |
| Annually | Rotate R2 API token and update `rclone.conf` on all hosts | Cloudflare R2 |
| Annually | Verify `rclone.conf` is on encrypted USB with Root CA backup | rclone / PKI |

---

## R2 Bucket Structure

After encryption, filenames in R2 will be unreadable. Through the rclone crypt remote, the logical structure is:

```
eirdom-backups/
  proxmox/
    vm-100/                          # EIRDOM-DC-01
      vzdump-qemu-100-2026_04_14-00_00_01.vma.zst
      vzdump-qemu-100-2026_04_14-00_00_01.log
      vzdump-qemu-100-2026_04_13-00_00_01.vma.zst
      ...
    vm-102/                          # EIRDOM-SUB-01
      vzdump-qemu-102-2026_04_13-00_00_01.vma.zst
      ...
    vm-120/                          # EIRDOM-WAZUH-01
      vzdump-qemu-120-2026_04_13-00_00_01.vma.zst
      ...
  docker/
    backup_20260414_010000/
      wordpress_db_20260414.sql.gz
      wordpress_files_20260414.tar.gz
      arr_configs_20260414.tar.gz
      traefik_20260414.tar.gz
    backup_20260413_010000/
      ...
```

---

## Cost Reduction Options

If you want to stay as close to free as possible, here are several levers:

| Option | Savings | Trade-off |
|--------|---------|-----------|
| Reduce daily retention from 7 to 3 days | ~55% less DC-01 storage | Shorter recovery window |
| Exclude EIRDOM-WAZUH-01 | ~16-24 GB less | Must rebuild Wazuh from scratch (config exportable separately) |
| Back up Wazuh config only (not full VM) | ~23 GB less | Faster rebuild but manual reinstall needed |
| Use R2 Infrequent Access tier | ~40% storage cost savings | 30-day minimum retention; retrieval fees apply |
| Weekly instead of daily for DC-01 | ~70% less DC-01 storage | Up to 7 days of AD changes at risk |

> **RECOMMENDATION:** Even at full retention (~75 GB), the monthly R2 cost is roughly $1.00. For the price of a cup of coffee, you get encrypted offsite disaster recovery for your entire infrastructure. The recommendation is to keep full retention and accept the nominal cost.