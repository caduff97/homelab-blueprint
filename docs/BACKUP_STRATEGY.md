# Backup Strategy

How I handle backups for my homelab. The goal is protecting irreplaceable data (photos, databases) and making disaster recovery straightforward.

## Current State

Backup strategy is fully active across two nodes.

| What | Status |
|------|--------|
| ZFS snapshots (tank pool) | ✅ Active — daily, 7-day retention |
| CT 100 vzdump → PBS | ✅ Active — daily midnight, wardstone PBS |
| Immich media backup | ✅ Active — nightly PBS backup with lean scope |
| Immich DB dump staging | ✅ Active — app-generated dump prepared before backup |
| CT 200 (hermes) → PBS | ✅ Active — nightly 1AM, wardstone PBS |
| VM 201 (spellcaster) → PBS | ✅ Active — nightly 1AM, wardstone PBS |
| TPT Center DB (Lightsail) → home pull | ✅ Active — nightly 23:57, folder rides CT 100 vzdump |
| Offsite / external backup | ❌ Not yet — all storage on-site |

## Storage Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│               proxmox (HP EliteDesk — Main Server)                   │
├─────────────────────────────────────────────────────────────────────┤
│  500GB NVMe → Proxmox OS + LXC root disk (~92GB via LVM)            │
│                                                                      │
│  ZFS pool "tank" (2× 500GB HDD stripe, ~928GB usable)               │
│  └── Immich library data (originals + backup staging)                │
│                                                                      │
│  ZFS snapshots: daily, 7-day retention (local only)                 │
│  125GB SSD → L2ARC read cache on tank pool                          │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│               wardstone (Dell desktop — Backup Node)                 │
├─────────────────────────────────────────────────────────────────────┤
│  240GB SSD → Proxmox OS + VM/LXC disks (local-lvm)                 │
│  500GB SSD → Additional VM/LXC storage (/mnt/storage)              │
│  1.8TB USB HDD → Primary PBS datastore (/mnt/pbs)                  │
└─────────────────────────────────────────────────────────────────────┘
```

## What's Being Backed Up

### ZFS Snapshots (active)

Daily snapshots of the entire `tank` pool via cron on the Proxmox host. Protects against accidental deletion and provides point-in-time recovery for Immich photos.

**Not a replacement for external backup** — snapshots live on the same disks as the data.

```bash
# /etc/cron.daily/zfs-snapshot
zfs snapshot tank@$(date +%Y-%m-%d)
zfs list -t snapshot -o name -s creation | grep "^tank@" | head -n -7 | xargs -r zfs destroy
```

### CT 100 vzdump → PBS (active)

Full LXC snapshot backup of the main Ubuntu container (all Docker services, configs). Runs nightly at midnight via Proxmox backup scheduler.

- **Storage:** wardstone PBS datastore `main`
- **Mode:** Snapshot (container frozen briefly, then backed up live)
- **Covers:** service folders under `/root/containers`
- **Note:** Bind-mounted paths (`/tank/immich`) are excluded — covered separately below

### Immich Media Backup (active)

The nightly Immich backup is intentionally lean: it focuses on the data that matters for recovery while avoiding bulky rebuildable artifacts that would just waste backup time and storage.

- **Included:** original media plus the app-generated backup folder prepared before the backup run
- **Excluded:** rebuildable derivatives such as thumbnails, transcoded video, and similar caches
- **Why this scope:** faster incrementals, less storage waste, and cleaner disaster recovery

This gives me the important user data and the database-dump artifacts together, without forcing the backup system to churn through everything Immich can regenerate on its own.

### Immich DB Dump Staging (active)

Before the nightly media backup, Immich creates its database dump so the backup window captures fresh DB state along with the rest of the important library data.

- **DB dump timing:** 12:00 AM
- **Backup timing:** 12:10 AM
- **Design goal:** let the dump finish first, then include that backup folder in the nightly run

Why this split:
- the DB snapshot is always created before the file backup starts
- the nightly backup captures media and DB-recovery artifacts together
- rebuildable data stays out of scope, keeping incrementals efficient

### CT 200 + VM 201 → PBS (active)

Hermes LXC and spellcaster Ubuntu VM backed up nightly at 1AM via Proxmox backup scheduler to wardstone PBS.

### TPT Center external app DB → home (active)

TPT Center is a public web app hosted off-site on AWS Lightsail (SQLite backed, Dockerized). It lives outside the cluster, so its database is pulled home nightly and then rides along in the normal CT 100 backup that already goes to PBS. This keeps Lightsail lightweight, keeps no home or PBS credentials on the cloud host, and reuses existing backup infrastructure instead of exposing PBS to the internet.

Flow, home pulls from cloud, never the reverse:

```
Lightsail (23:55 MST)                CT 100 home (23:57 MST)                   wardstone PBS
  generate verified SQLite  ── ssh/rsync pull (read-only key) ──▶  /root/backups/tpt-center/sqlite/  ──▶ nightly CT 100 vzdump
  .db.gz, 14-day retention                                           30-day local retention
```

- **Cloud side (Lightsail):** systemd timer `tpt-center-db-backup.timer` runs the `deploy` user script `/home/deploy/scripts/tpt-center-backup-sqlite.sh` at 23:55 MST. It uses the SQLite backup API from inside the backend container (consistent while the app runs), runs `PRAGMA integrity_check`, gzips the result, and keeps 14 days locally in `/home/deploy/backups/tpt-center/sqlite/`.
- **Home side (CT 100):** systemd timer `tpt-center-pull-backup.timer` runs `/root/scripts/tpt-center-pull-backup.sh` at 23:57 MST. It rsyncs the `.db.gz` files over SSH using a dedicated read-only key (`/root/.ssh/tpt_lightsail_backup`), verifies gzip integrity, and keeps 30 days in `/root/backups/tpt-center/sqlite/`.
- **PBS coverage:** that folder sits inside CT 100, so the existing nightly CT 100 vzdump captures it automatically. No new PBS job is required.

Security notes:
- Home pulls from Lightsail. Lightsail holds no key into the home network and no PBS credentials.
- The pull key is dedicated to this task; if Lightsail is ever compromised it cannot reach or delete PBS backups.
- Do not back up the live SQLite file directly; the SQLite backup API produces a consistent copy while the app is writing.

### Restore TPT Center database

```bash
# On CT 100 (home), pick a backup and decompress
cd /root/backups/tpt-center/sqlite
gunzip -c tpt-center-YYYYMMDD-HHMMSS.db.gz > /tmp/tpt-restore.db
sqlite3 /tmp/tpt-restore.db "PRAGMA integrity_check;"   # expect: ok
# Copy /tmp/tpt-restore.db back to Lightsail and swap it in as /data/app.db for the backend container.
```

## Schedule

| Time | Job | Node | Status |
|------|-----|------|--------|
| Daily (cron) | ZFS snapshot → tank pool | proxmox | ✅ Active |
| 12:00 AM | CT 100 vzdump → wardstone PBS | proxmox | ✅ Active |
| 12:00 AM | CT 200 + VM 201 vzdump → PBS | wardstone | ✅ Active |
| 12:00 AM | Immich DB dump staging | main app host | ✅ Active |
| 12:10 AM | Immich media backup | main app host / PBS flow | ✅ Active |
| 11:55 PM | TPT Center DB backup (Lightsail, cloud) | AWS Lightsail | ✅ Active |
| 11:57 PM | TPT Center DB pull → CT 100 | proxmox | ✅ Active |

## Retention Policy

| Snapshots | Count |
|-----------|-------|
| Daily | 7 |
| Weekly | 4 |
| Monthly | 3 |

~3 months of point-in-time recovery.

## Why the Immich backup scope is selective

The goal is not to back up every byte Immich touches. The goal is to protect what would actually hurt to lose.

That means:
- keep originals
- keep the backup folder that contains the fresh DB dump
- skip generated derivatives Immich can rebuild

This keeps the nightly job practical and avoids wasteful rescans when only a small amount of meaningful data changed.

## Recovery Procedures

### Restore LXC or VM from PBS

In Proxmox web UI: **wardstone → PBS storage → select snapshot → Restore**

Or via CLI on the target node:
```bash
qmrestore /path/to/backup.vma <vmid>   # VM
pct restore <ctid> /path/to/backup.tar # LXC
```

### Restore Immich media from PBS

```bash
proxmox-backup-client restore immich-photos.pxar:/restore-path \
  --repository "root@pam@wardstone-ip:main" \
  --backup-type host \
  --backup-id immich-photos
```

Restore the important library data first, including the backup artifacts created before the nightly run.

### Restore Immich database

```bash
# Use the database dump captured in the backup folder from the same nightly backup set.
# Restore DB first, then bring the full Immich stack back.
```

### Restore from ZFS snapshot

```bash
# List available snapshots
zfs list -t snapshot

# Roll back pool to a snapshot (DESTRUCTIVE — loses changes after snapshot)
zfs rollback tank@2026-05-26

# Restore a single file/folder non-destructively
cp /tank/.zfs/snapshot/2026-05-26/immich/uploads/path/to/file /tank/immich/uploads/
```

### Verify ZFS health

```bash
zpool status tank
zpool scrub tank   # triggers integrity check — run monthly
```

## 3-2-1 Rule Status

| Rule | Status |
|------|--------|
| **3** copies of data | ✅ Live (ZFS) + PBS on wardstone + ZFS snapshots |
| **2** different media | ✅ NVMe/HDD on proxmox + USB HDD on wardstone |
| **1** offsite copy | ❌ Not yet — all storage on-site |

**Next milestone**: offsite backup — cloud storage (Backblaze B2 via Restic) or a physically separate location.
