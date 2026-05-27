# Backup Strategy

How I handle backups for my homelab. The goal is protecting irreplaceable data (photos, databases) and making disaster recovery straightforward.

## Current State

Migration to HP EliteDesk is complete. The backup strategy is partially active — ZFS snapshots are running, Restic is paused pending a new destination (PBS on black-hawk, which is being repurposed).

| What | Status |
|------|--------|
| ZFS snapshots (tank pool) | ✅ Active — daily, 7-day retention |
| Restic (databases, configs) | ⏸ Paused — needs new destination (PBS on black-hawk) |
| PBS on black-hawk | 🔜 Planned — black-hawk being converted to Proxmox |
| Offsite / external backup | 🔜 Planned — 1.8TB HDD on black-hawk |

## Storage Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│               HP EliteDesk (Main Server — Proxmox)                   │
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
│               black-hawk (Dell — being repurposed)                   │
├─────────────────────────────────────────────────────────────────────┤
│  240GB SSD → Proxmox OS + VM/LXC disks                              │
│  500GB SSD → PBS datastore (secondary / VM storage)                 │
│  1.8TB HDD → Primary PBS datastore (~1.2TB free after backups)      │
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

### Restic — databases and configs (paused)

Was backing up `~/containers/` (all Docker data, Postgres DBs) to a local SSD. That SSD has since been removed. Restic repo lives at `~/backup/restic` — intact but not being updated.

**Needs**: PBS set up on black-hawk, then Restic destination updated to push there.

### Planned: PBS on black-hawk

Once black-hawk is running Proxmox + PBS:

| What | Source | Destination | Method |
|------|--------|-------------|--------|
| LXC 100 full backup | HP EliteDesk | black-hawk PBS (1.8TB) | vzdump |
| Immich photos | /tank/immich/uploads | black-hawk PBS (1.8TB) | rsync or PBS |
| Databases/configs | ~/containers/ | black-hawk PBS (1.8TB) | Restic |

## Schedule

| Time | Job | Status |
|------|-----|--------|
| Daily (cron) | ZFS snapshot → tank pool | ✅ Active |
| 3:00 AM | Restic → databases/configs | ⏸ Paused |
| TBD | vzdump → PBS on black-hawk | 🔜 Planned |
| TBD | rsync/PBS → photos to black-hawk | 🔜 Planned |

## Retention Policy

| Snapshots | Count |
|-----------|-------|
| Daily | 7 |
| Weekly | 4 |
| Monthly | 3 |

~3 months of point-in-time recovery for database data.

## Why Restic for databases, rsync for photos

| | Restic | rsync |
|---|--------|-------|
| Point-in-time recovery | Yes — go back weeks | No — last sync only |
| Encryption at rest | Yes | No |
| Speed on large data | Slow | Very fast |
| Browse files without tools | No | Yes |
| Catches silent DB corruption | Yes (old snapshots survive) | No |

Databases need Restic because corruption can go unnoticed for days — rsync would silently overwrite the last good backup before you notice. Photos don't have this problem.

## Recovery Procedures

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
| **3** copies of data | ⏸ 1 live (ZFS) + local snapshots only — external pending |
| **2** different media | 🔜 Pending PBS setup on black-hawk |
| **1** offsite copy | ❌ Not yet — all storage on-site |

**Next milestone**: PBS on black-hawk → achieve 2-copy minimum and re-enable Restic.
