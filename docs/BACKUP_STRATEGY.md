# Backup Strategy

How I handle backups for my homelab. The goal is protecting irreplaceable data (photos, databases) and making disaster recovery straightforward.

## Current State

Backup strategy is fully active across two nodes.

| What | Status |
|------|--------|
| ZFS snapshots (tank pool) | ✅ Active — daily, 7-day retention |
| CT 100 vzdump → PBS | ✅ Active — daily midnight, wardstone PBS |
| Immich photos → PBS | ✅ Active — nightly 2AM, proxmox-backup-client |
| CT 200 (hermes) → PBS | ✅ Active — nightly 1AM, wardstone PBS |
| VM 201 (spellcaster) → PBS | ✅ Active — nightly 1AM, wardstone PBS |
| Offsite / external backup | ❌ Not yet — all storage on-site |

## Storage Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│               proxmox (HP EliteDesk — Main Server)                   │
├─────────────────────────────────────────────────────────────────────┤
│  500GB NVMe → Proxmox OS + LXC root disk (~92GB via LVM)            │
│                                                                      │
│  ZFS pool "tank" (2× 500GB HDD stripe, ~928GB usable)               │
│  └── /tank/immich/uploads/   ~548GB  ← Immich photos (originals)    │
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
- **Note:** Bind-mounted paths (`/tank/immich`) are excluded — covered separately below

### Immich Photos → PBS (active)

`proxmox-backup-client` backs up `/tank/immich/uploads` directly to wardstone PBS. After the first full backup (~510GB), subsequent runs only upload changed chunks — typically fast.

```bash
# /usr/local/bin/immich-backup.sh (runs via /etc/cron.d/pbs-immich at 2AM)
PBS_FINGERPRINT="..." PBS_PASSWORD="..." \
proxmox-backup-client backup immich-photos.pxar:/tank/immich/uploads \
  --repository "root@pam@wardstone-ip:main" \
  --backup-type host \
  --backup-id immich-photos
```

### CT 200 + VM 201 → PBS (active)

Hermes LXC and spellcaster Ubuntu VM backed up nightly at 1AM via Proxmox backup scheduler to wardstone PBS.

## Schedule

| Time | Job | Node | Status |
|------|-----|------|--------|
| Daily (cron) | ZFS snapshot → tank pool | proxmox | ✅ Active |
| 12:00 AM | CT 100 vzdump → wardstone PBS | proxmox | ✅ Active |
| 12:00 AM | CT 200 + VM 201 vzdump → PBS | wardstone | ✅ Active |
| 2:00 AM | Immich photos → wardstone PBS | proxmox | ✅ Active |

## Retention Policy

| Snapshots | Count |
|-----------|-------|
| Daily | 7 |
| Weekly | 4 |
| Monthly | 3 |

~3 months of point-in-time recovery.

## Why proxmox-backup-client for photos

vzdump excludes bind-mounted paths like `/tank/immich` — Proxmox explicitly warns about this during backup. `proxmox-backup-client` backs up arbitrary directories directly to PBS with full deduplication, so only changed chunks upload after the first run.

## Recovery Procedures

### Restore LXC or VM from PBS

In Proxmox web UI: **wardstone → PBS storage → select snapshot → Restore**

Or via CLI on the target node:
```bash
qmrestore /path/to/backup.vma <vmid>   # VM
pct restore <ctid> /path/to/backup.tar # LXC
```

### Restore Immich photos from PBS

```bash
proxmox-backup-client restore immich-photos.pxar:/restore-path \
  --repository "root@pam@wardstone-ip:main" \
  --backup-type host \
  --backup-id immich-photos
```

### Restore Immich database

```bash
# 1. Dump from running postgres (while healthy):
docker exec -t immich_postgres pg_dumpall -c -U postgres > immich_dump.sql

# 2. Stop Immich, restore:
cd ~/containers/immich
docker compose stop immich-server immich-machine-learning
docker compose up -d database
# wait for healthy...
docker exec -i immich_postgres psql -U postgres -d postgres < immich_dump.sql
docker compose up -d
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
