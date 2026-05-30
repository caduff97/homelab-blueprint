# Homelab Infrastructure

This is my homelab setup — a two-node Proxmox cluster running multiple services for personal use and hosting web applications. I built it with one goal: **keep things secure without making my life complicated**.

The result? Zero port forwarding on my router, my home IP completely hidden from the internet, and I can access my personal services from anywhere through a zero-trust VPN. All of this runs on repurposed hardware.

## The Problem I Solved

When I started self-hosting, I faced the classic homelab dilemmas:

- **Port forwarding feels risky** - Opening ports exposes your home IP and attack surface
- **Dynamic IP is annoying** - My ISP changes my IP randomly
- **VPN for everything is overkill** - I don't want to VPN just to show someone a web app
- **SSL certificates are a pain** - Let's Encrypt works, but managing renewals across services gets old

My solution splits traffic into two paths: **public** (through Cloudflare Tunnel) and **private** (through Tailscale). Each handles its own SSL, and I never touch my router's port forwarding settings.

## How It Works

```
                              INTERNET
                                 │
            ┌────────────────────┴────────────────────┐
            ▼                                         ▼
   ┌─────────────────────┐               ┌─────────────────────┐
   │   CLOUDFLARE EDGE   │               │     TAILSCALE       │
   │                     │               │                     │
   │  - Handles SSL      │               │  - WireGuard VPN    │
   │  - DDoS protection  │               │  - Device auth      │
   │  - Hides my IP      │               │  - Auto mesh        │
   └──────────┬──────────┘               └──────────┬──────────┘
              │                                     │
              ▼                                     ▼
   ┌─────────────────────┐               ┌─────────────────────┐
   │    cloudflared      │               │   Caddy Proxy       │
   │    (Docker)         │               │   (*.internal)      │
   └──────────┬──────────┘               └──────────┬──────────┘
              │                                     │
        ┌─────┴─────┐                         ┌─────┴─────┐
        ▼           ▼                         ▼           ▼
   ┌─────────┐ ┌─────────┐             ┌─────────┐ ┌─────────┐
   │ Web App │ │ Web App │             │ Immich  │ │Nextcloud│
   │   #1    │ │   #2    │             │ Photos  │ │  Files  │
   └─────────┘ └─────────┘             └─────────┘ └─────────┘
        PUBLIC SERVICES                    PRIVATE SERVICES
      (anyone can access)              (only my devices)
```

## What I'm Running

### Public Services
Web applications accessible to anyone - client projects, side projects, whatever needs to be on the internet. Cloudflare handles SSL and hides my home IP completely.

### Private Services
- **Immich** - Self-hosted Google Photos replacement
- **Nextcloud** - File sync across all my devices
- **CapitalLog** - Personal investments manager (own project)
- **Uptime Kuma** - Monitoring dashboard with alerts
- **DailyTxt** - Private daily journal
- **Dockge** - Stack management UI

### AI & Automation
- **Hermes Agent** - AI agent accessible via Telegram — remote homelab control and task automation

## The Hardware

Two-node Proxmox cluster on repurposed hardware:

### Node 1 — proxmox (HP EliteDesk mini PC)
| Component | Spec | Purpose |
|-----------|------|---------|
| CPU | Intel Core i5-6500 (4 cores) | |
| RAM | 16GB DDR4 | |
| System | 500GB NVMe | Proxmox OS + LXC root disk |
| Data | 2× 500GB HDD | ZFS stripe pool "tank" (~928GB usable) |
| Cache | 125GB SSD | ZFS L2ARC on tank pool |

### Node 2 — wardstone (Dell desktop)
| Component | Spec | Purpose |
|-----------|------|---------|
| CPU | Intel Core i5-4500 (4 cores) | |
| RAM | 32GB DDR4 | |
| System | 240GB SSD | Proxmox OS + VM/LXC local disks (local-lvm) |
| Storage | 500GB SSD | Additional VM/LXC storage (/mnt/storage) |
| Backup | 1.8TB USB HDD | PBS primary datastore (/mnt/pbs) |

Total cost: **~$53** for the HP EliteDesk ($25 motherboard, $10 SSD, $6 HDDs, $12 cables + thermal paste). Wardstone was a repurposed machine already on hand.

### Cluster Overview

| Machine | Role | Status |
|---------|------|--------|
| proxmox (HP EliteDesk) | Main server — all Docker services | **Active** |
| wardstone (Dell desktop) | Backup server + daily workstation | **Active** |

## Security Model

The approach is simple:
1. **Router has zero open ports** - All tunnel connections go outbound
2. **Private services bind to localhost** - Can't reach them even on LAN
3. **Tailscale authenticates devices** - Not just network access, device identity

## Documentation

- **[Architecture Guide](docs/ARCHITECTURE.md)** - Technical details, cluster setup, and how to add new services
- **[Backup Strategy](docs/BACKUP_STRATEGY.md)** - How backups work, what's covered, retention policies
- **[Lessons Learned](docs/LESSONS_LEARNED.md)** - Mistakes I made so you don't have to

## Quick Commands

```bash
# Check if tunnel is healthy
docker logs cloudflared --tail 20

# Container status
docker ps --format "table {{.Names}}\t{{.Status}}"

# Reload Caddy after Caddyfile changes
cd ~/containers/caddy && docker compose up -d --force-recreate

# Check backup status (run on proxmox host)
tail -20 /var/log/immich-backup.log

# PBS backup status
# Open https://wardstone-ip:8007 → Datastore → main → Content
```

## Adding New Services

**Want to expose something publicly?**
1. Deploy your container
2. Add its network to cloudflared
3. Add hostname in Cloudflare Zero Trust dashboard
4. Done - SSL and DNS handled automatically

**Want something private?**
1. Deploy container bound to `127.0.0.1:PORT`
2. Add to `~/containers/caddy/Caddyfile`:
   ```
   myservice.internal {
     reverse_proxy 127.0.0.1:PORT
   }
   ```
3. Reload Caddy: `cd ~/containers/caddy && docker compose up -d --force-recreate`
4. Access at `https://myservice.internal`

That's it. Clean URL, valid cert, no port numbers.

---

Built because I wanted to own my data without the headache. If this inspires you to build your own, that's awesome.
