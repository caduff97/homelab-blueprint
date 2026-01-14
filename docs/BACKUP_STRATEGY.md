# Backup Strategy

How I handle backups for my homelab. The goal is protecting irreplaceable data (photos, documents) and making disaster recovery possible.

## Current State

**Status**: Local backups operational, offsite in progress

| What | Status |
|------|--------|
| Database backups | Automated daily |
| Nextcloud files | Automated daily |
| Config files | Automated daily |
| Photo library (360GB) | NOT backed up yet |
| Offsite copy | Planned |

## Storage Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           STORAGE LAYOUT                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────┐     ┌──────────────────────┐                  │
│  │   240GB SSD          │     │  120GB SSD           │                  │
│  │   (System Drive)     │     │  /mnt/backup         │                  │
│  ├──────────────────────┤     ├──────────────────────┤                  │
│  │ /         ~80GB free │     │  Restic repository   │                  │
│  │ /home     ~75GB free │────▶│  ~20GB used          │                  │
│  │                      │     │  ~90GB free          │                  │
│  │ Contains:            │     │                      │                  │
│  │ - Docker containers  │     │  Daily backups       │                  │
│  │ - Postgres DBs       │     │  Email alerts        │                  │
│  │ - App configs        │     │  Auto-prune          │                  │
│  └──────────────────────┘     └──────────────────────┘                  │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │              2TB External HDD (USB)                               │   │
│  │              /mnt/data - ~700GB used                              │   │
│  ├──────────────────────────────────────────────────────────────────┤   │
│  │  /mnt/data/immich/     ~360GB  ← PHOTOS (NO BACKUP!)             │   │
│  │    - 42,000+ photos                                               │   │
│  │    - 7,000+ videos                                                │   │
│  │  /mnt/data/nextcloud/  ~800MB  ← Files (BACKED UP)               │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

The elephant in the room: my 360GB photo library has zero redundancy. If that external HDD fails, everything is gone. Solving this is my top priority.

## What's Being Backed Up

| Path | Contents | Size |
|------|----------|------|
| `~/containers/` | All Postgres databases, app data | ~2GB |
| `/mnt/data/nextcloud/` | Nextcloud user files | ~800MB |
| `~/pihole/` | Pi-Hole configuration | ~50MB |

**Total backup size**: ~19GB (with Restic deduplication)

## Backup Schedule

| When | What | Notification |
|------|------|--------------|
| Daily 3:00 AM | Full incremental backup | Email on completion |

## Retention Policy

Restic handles this automatically:

- **7 daily** snapshots
- **4 weekly** snapshots
- **3 monthly** snapshots

This gives me about 3 months of point-in-time recovery options.

## Why Restic?

I chose Restic over alternatives because:

1. **Deduplication** - Only stores changes, not full copies each time
2. **Encryption** - Data encrypted at rest, safe for cloud storage later
3. **Incremental** - Fast daily backups after initial seed
4. **Simple restore** - Can restore entire snapshots or specific paths

## Backup Script

The backup runs via cron and sends email notifications:

```bash
#!/bin/bash
# Simplified example - actual script has more logging

export RESTIC_REPOSITORY="/mnt/backup/restic"
export RESTIC_PASSWORD_FILE="/etc/restic-password"

# Run backup
restic backup \
  ~/containers \
  /mnt/data/nextcloud \
  ~/pihole

# Prune old snapshots
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 3 --prune

# Send notification
echo "Backup completed" | mail -s "Homelab Backup" my@email.com
```

## Recovery Procedures

### Restore Entire Snapshot

```bash
export RESTIC_REPOSITORY="/mnt/backup/restic"
export RESTIC_PASSWORD_FILE="/etc/restic-password"

# List available snapshots
sudo -E restic snapshots

# Restore specific snapshot
sudo -E restic restore abc123 --target /path/to/restore
```

### Restore Specific Path

```bash
# Restore just Pi-Hole config from latest backup
sudo -E restic restore latest --target /tmp/restore --include "pihole"
```

### Verify Backup Integrity

```bash
sudo -E restic check
```

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| External HDD failure | **CATASTROPHIC** - Lose 42K photos | None yet |
| Database corruption | High - Lose app data | Daily Restic backups |
| Accidental deletion | High - Lose files | 3 months of snapshots |
| Ransomware | Catastrophic | Partial (local only, no offsite) |
| Server total loss | High | Can restore from backup SSD |

## What's NOT Backed Up (Yet)

| Data | Size | Priority | Notes |
|------|------|----------|-------|
| Immich photos | ~360GB | **CRITICAL** | Too large for backup SSD |
| Uptime Kuma | ~8MB | Low | Can recreate monitoring config |
| Portainer | ~10MB | Low | Stack configs in git anyway |
| System configs | <1MB | Medium | `/etc/fstab`, crontabs, etc. |

## The 3-2-1 Goal

I'm working toward the 3-2-1 backup rule:

| Rule | Current | Target |
|------|---------|--------|
| **3** copies of data | 2 (production + local SSD) | 3 |
| **2** different media | 2 (SSD + HDD) | 2 |
| **1** offsite | 0 | 1 |

### Offsite Options I'm Considering

**Option A: Backblaze B2 (free tier)**
- Pros: 10GB free, automatic offsite, easy Restic integration
- Cons: Only fits databases, not photos
- Best for: Critical database backups

**Option B: Second 2TB HDD (~$70)**
- Pros: Fits entire photo library, one-time cost
- Cons: Manual rotation, need offsite storage location
- Best for: Full Immich backup with offsite rotation

**Option C: Hybrid (likely winner)**
- Backblaze B2 for databases (free tier)
- Second HDD for photos, rotated offsite monthly

## Lessons Learned

1. **Start with databases** - They're small and critical. Don't wait for the "perfect" solution.

2. **Test restores** - A backup that can't be restored isn't a backup. I test mine quarterly.

3. **Automate everything** - Manual backups don't happen. Cron + email notifications = peace of mind.

4. **Separate backup drive** - Keeping backups on the same drive as production is asking for trouble.

5. **Encryption from day one** - Using Restic's encryption means I can safely move backups to cloud storage later.
