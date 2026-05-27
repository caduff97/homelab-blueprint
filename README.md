# Homelab Infrastructure

This is my homelab setup - a single server running multiple services for personal use and hosting web applications. I built it with one goal: **keep things secure without making my life complicated**.

The result? Zero port forwarding on my router, my home IP completely hidden from the internet, and I can access my personal services from anywhere through a zero-trust VPN. All of this runs on repurposed hardware that cost me nothing.

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

## The Hardware

Repurposed hardware — the main server is an HP EliteDesk mini PC running Proxmox:

| Component | Spec | Purpose |
|-----------|------|---------|
| CPU | Intel Core i5-6500 (4 cores) | |
| RAM | 16GB DDR4 | |
| System | 500GB NVMe | Proxmox OS + LXC root disk |
| Data | 2× 500GB HDD | ZFS stripe pool "tank" (~928GB usable) |
| Cache | 125GB SSD | ZFS L2ARC on tank pool |

Total cost: **~$53** ($25 motherboard, $10 SSD, $6 HDDs, $12 cables + thermal paste)

All service data lives on the ZFS pool. ZFS handles daily snapshots (7-day retention) for local point-in-time recovery.

### Migration Status

Full infrastructure migration from old Dell desktop to HP EliteDesk:

| Machine | Role | Status |
|---------|------|--------|
| HP EliteDesk | Main server — Proxmox host | **Active** |
| Dell desktop | Pending retirement → future Proxmox Backup Server | In transition |
| Netgear ReadyNAS Duo V2 | Retired — drives repurposed into ZFS pool on HP EliteDesk | Done |

## Why This Setup?

| Old Way | My Way |
|---------|--------|
| Port forward 80/443 | Cloudflare Tunnel (outbound only) |
| Expose home IP | IP completely hidden |
| DDNS for dynamic IP | Cloudflare handles DNS |
| Manual SSL certs | Automatic from both providers |
| OpenVPN server | Tailscale (zero config) |
| VPN for everything | Only private stuff needs VPN |

The security model is simple:
1. **Router has zero open ports** - All tunnel connections go outbound
2. **Private services bind to localhost** - Can't reach them even on LAN
3. **Tailscale authenticates devices** - Not just network access, device identity

## Documentation

I've documented everything so I can recreate this setup if needed (and maybe it helps someone else):

- **[Architecture Guide](docs/ARCHITECTURE.md)** - Technical details and how to add new services
- **[Backup Strategy](docs/BACKUP_STRATEGY.md)** - How backups work, what's covered, what's not
- **[Lessons Learned](docs/LESSONS_LEARNED.md)** - Mistakes I made so you don't have to
- **[black-hawk Setup](docs/BLACK_HAWK_SETUP.md)** - Plan for repurposing old Dell as dev machine + backup server

## Quick Commands

```bash
# Check if tunnel is healthy
docker logs cloudflared --tail 20

# Container status
docker ps --format "table {{.Names}}\t{{.Status}}"

# Reload Caddy after Caddyfile changes
cd ~/containers/caddy && docker compose up -d --force-recreate

# Run a backup manually
sudo /path/to/backup.sh

# Check backup history
sudo -E restic snapshots
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
