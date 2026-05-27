# Architecture Guide

This document covers the technical implementation details. For the overview and rationale, see the [README](../README.md).

## Network Interfaces

The server is a Proxmox LXC container (CT 100) bridged to the home LAN via the Proxmox host.

| Interface | Where | Purpose |
|-----------|-------|---------|
| `vmbr0` | Proxmox host | Linux bridge — connects LXC to physical NIC |
| `eth0` | LXC container | LAN access |
| `tailscale0` | LXC container | VPN mesh — personal devices only |
| `docker0` + named bridges | LXC container | Docker networking |

Private services bind to `127.0.0.1` inside the LXC, so they're unreachable from the LAN directly — only accessible through Tailscale.

## Cloudflare Tunnel Setup

### How It Works

The `cloudflared` container maintains an outbound connection to Cloudflare's edge. When traffic hits my domain, Cloudflare routes it through this tunnel to my container - no inbound ports needed.

### Docker Compose

```yaml
services:
  cloudflared:
    container_name: cloudflared
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}
    networks:
      - app1_default
      - app2_default
      # Add each app's network here

networks:
  app1_default:
    external: true
  app2_default:
    external: true
```

The key insight: **cloudflared must join each application's Docker network** to reach containers by their service name. Otherwise it can't resolve `http://myapp-frontend:80`.

### Adding a Public Service

1. Deploy the app with Docker Compose (creates its own network like `myapp_default`)
2. Set an explicit `container_name` on the service that cloudflared will route to — without it, Compose appends `-1` and the tunnel config becomes fragile
3. Add the network to `~/containers/cloudflared/docker-compose.yml` and redeploy: `docker compose up -d`
4. In Cloudflare Zero Trust dashboard:
   - Go to Networks → Tunnels → my tunnel → Public Hostname
   - Add hostname: `myapp.mydomain.com`
   - Service: `http://myapp-container:80` (use the exact `container_name` value)
5. DNS record is created automatically

### Hosting Non-Docker Services

Sometimes I want to expose something running directly on the host (like a dev server). Cloudflared needs to reach the host network:

```yaml
services:
  cloudflared:
    # ... existing config ...
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

Then in the tunnel config, use `http://host.docker.internal:PORT` as the service URL.

## Private Service Access

### How It Works

Private services bind to `127.0.0.1:PORT` and are accessed via three components working together:

1. **dnsmasq** — resolves `*.internal` to the server's Tailscale IP
2. **Tailscale split DNS** — routes `internal` domain queries to dnsmasq for every tailnet device
3. **Caddy** — reverse proxy in host network mode, routes by hostname to the right `127.0.0.1:PORT`

Result: every service gets a clean URL like `https://myservice.internal` with a valid cert — no ports, no browser warnings.

### Caddy

Runs in `network_mode: host` so it can reach all `127.0.0.1:PORT` bindings directly.

```yaml
services:
  caddy:
    image: caddy:2
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./data:/data
```

Caddy uses its built-in local CA (`local_certs`) to issue certificates for `*.internal` automatically. No external dependencies or renewal management needed.

```
{
  local_certs
}

myservice.internal {
  reverse_proxy 127.0.0.1:PORT
}
```

### dnsmasq

Single-line config resolves all `*.internal` to the server's Tailscale IP:

```
address=/.internal/<server-tailscale-ip>
no-resolv
server=1.1.1.1
server=8.8.8.8
```

### Tailscale Split DNS

In Tailscale admin → DNS → Add nameserver → Custom:
- IP: server's Tailscale IP
- Restrict to domain: `internal`

This makes `*.internal` resolve correctly on every tailnet device without changing their local DNS settings.

### Adding a Private Service

1. Bind the container to localhost:
   ```yaml
   ports:
     - "127.0.0.1:PORT:PORT"
   ```

2. Add an entry to `~/containers/caddy/Caddyfile`:
   ```
   myservice.internal {
     reverse_proxy 127.0.0.1:PORT
   }
   ```

3. Reload Caddy: `cd ~/containers/caddy && docker compose up -d --force-recreate`

4. Access at `https://myservice.internal`

### Trusting the Local CA

Caddy's root CA lives at `~/containers/caddy/data/caddy/pki/authorities/local/root.crt`. Install it once per device — after that all current and future `*.internal` services work without warnings.

**macOS**:
```bash
scp root@<server-ip>:~/containers/caddy/data/caddy/pki/authorities/local/root.crt ~/Desktop/caddy-root.crt
open ~/Desktop/caddy-root.crt
```
In Keychain Access: double-click the certificate → Trust → **Always Trust** → close.

**iOS/iPadOS**: AirDrop the `.crt` file → tap to install → Settings → General → About → Certificate Trust Settings → enable full trust.

**Android**: Settings → Security → Install certificate → CA Certificate.

**Docker containers** (e.g. Uptime Kuma monitoring internal services):
```yaml
volumes:
  - /root/containers/caddy/data/caddy/pki/authorities/local/root.crt:/usr/local/share/ca-certificates/caddy-root.crt:ro
environment:
  - NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/caddy-root.crt
```

### Why Localhost Binding Matters

By binding to `127.0.0.1` instead of `0.0.0.0`, services are unreachable from the LAN directly. Only Caddy (running in host network mode on the same machine) can reach them.

## DNS Architecture

### The Flow

```
My phone/laptop (on Tailscale)
        │
        ▼
Tailscale Split DNS
        │
        ├──► *.internal → dnsmasq (server's Tailscale IP:53)
        │         └──► Resolves to server IP → Caddy → service
        │
        └──► Everything else → 1.1.1.1 / 8.8.8.8
```

### Configuration

1. **dnsmasq container** on the server: resolves `*.internal` → server's Tailscale IP
2. **Tailscale admin** → DNS → Add nameserver → Custom → server's Tailscale IP, restricted to domain `internal`

No global DNS override — only `*.internal` queries are intercepted. All other DNS resolves normally through upstream providers.

## Proxmox + LXC Setup (New Infrastructure)

The main server runs Proxmox VE as a Type-1 hypervisor with a single Ubuntu LXC container hosting all Docker services.

### Why LXC over VM

LXC shares the Proxmox kernel directly — ~50MB RAM overhead vs ~500MB for a full VM. On 16GB RAM with many Docker containers, that headroom matters.

### LXC Requirements for Docker

Two settings must be enabled on the container before starting it (CT → Options → Features):
- **Nesting** — allows Docker to create namespaces inside the container
- **Keyctl** — required by some container runtimes

### LXC Requirements for Tailscale (TUN device)

Tailscale needs a TUN network device, which isn't available in LXC by default. Proxmox's UI doesn't expose this option, so it must be added manually on the Proxmox host:

```bash
# Load TUN module on the host (persistent across reboots)
modprobe tun
echo "tun" >> /etc/modules

# Add TUN device passthrough to the LXC config
echo 'lxc.cgroup2.devices.allow: c 10:200 rwm' >> /etc/pve/lxc/100.conf
echo 'lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file' >> /etc/pve/lxc/100.conf

# Restart the container
pct stop 100 && pct start 100
```

### Proxmox Repository Setup

Proxmox defaults to enterprise repositories that require a paid subscription. For homelab use, disable them and add the free repos:

```bash
# Disable enterprise repos (they use .sources format, not .list)
echo "Enabled: no" >> /etc/apt/sources.list.d/pve-enterprise.sources
echo "Enabled: no" >> /etc/apt/sources.list.d/ceph.sources

# Add no-subscription repos
cat > /etc/apt/sources.list.d/pve-no-subscription.list << 'EOF'
deb http://download.proxmox.com/debian/pve trixie pve-no-subscription
deb http://download.proxmox.com/debian/ceph-squid trixie no-subscription
EOF

apt-get update
```

The web UI shows a "No valid subscription" popup on login — dismiss it, everything works fine.

### Storage Layout

| Mount | Drive | Purpose |
|-------|-------|---------|
| Proxmox system | 500GB NVMe | Proxmox OS + LXC root disk (64GB via LVM) |
| ZFS pool `tank` | 2× 500GB HDD (stripe) | All service data — ~928GB usable |
| ZFS L2ARC | 125GB SSD | Read cache for tank pool |

The two HDDs came from a retired Netgear ReadyNAS Duo V2. The ZFS pool is registered as a Proxmox Directory storage ("tank"), making it visible in the UI and available for future VM/LXC volumes.

The LXC accesses data via a bind mount of `/tank/immich` into the container, configured in `/etc/pve/lxc/100.conf`:

```
mp0: /tank/immich,mp=/tank/immich
```

Inside the LXC, service data is organized as:
```
/tank/immich/
└── uploads/    ← UPLOAD_LOCATION (photos, originals)
```

### ZFS Snapshots

Daily snapshots via cron on the Proxmox host, 7-day retention:

```bash
# /etc/cron.daily/zfs-snapshot
zfs snapshot tank@$(date +%Y-%m-%d)
zfs list -t snapshot -o name -s creation | grep "^tank@" | head -n -7 | xargs -r zfs destroy
```

Monitor pool health:
```bash
zpool status tank
zpool list tank
zfs list -t snapshot
```

## What's NOT Exposed

- No 80/443 - Cloudflare Tunnel handles this
- No VPN ports - Tailscale uses NAT traversal
- Private service ports - bound to localhost, only Caddy reaches them

## CI/CD — Auto-Deploy

### How It Works

Push to `main` → GitHub sends a webhook POST to `https://webhooks.carlostri.com/hooks/project-name` → `adnanh/webhook` validates the HMAC signature → runs `deploy.sh project-name`.

```
push to main
     │
     ▼
GitHub webhook POST (HMAC-signed)
     │
     ▼
webhooks.carlostri.com → cloudflared → host:9000
     │
     ▼
webhook service validates signature → deploy.sh project-name
     │
     ▼
git pull → docker compose up --build -d
```

No GitHub Actions runners. No CI infrastructure on GitHub. Just one lightweight systemd service on the server.

### Webhook Service

Installed via apt (`webhook` package), runs as a systemd service. Config lives at `/root/webhook/` alongside the deploy script:

```
/root/webhook/
├── hooks.yaml    ← one entry per project
└── deploy.sh     ← generic deploy script
```

Systemd drop-in at `/etc/systemd/system/webhook.service.d/override.conf` points the service to `/root/webhook/hooks.yaml`.

Logs: `journalctl -u webhook -f`

### Deploy Script

```bash
#!/bin/bash
set -e

PROJECT="$1"

if [[ -z "$PROJECT" || "$PROJECT" =~ [/\.] ]]; then
  echo "Usage: deploy.sh <project>"
  exit 1
fi

DIR="/root/containers/$PROJECT"

if [ ! -d "$DIR" ]; then
  echo "Project not found: $PROJECT"
  exit 1
fi

cd "$DIR"
git pull
docker compose up --build -d
```

### hooks.yaml Structure

Each entry validates the GitHub HMAC signature and checks the branch before triggering a deploy:

```yaml
- id: project-name
  execute-command: /root/webhook/deploy.sh
  pass-arguments-to-command:
    - source: string
      name: project-name
  trigger-rule:
    and:
      - match:
          type: payload-hmac-sha256
          secret: YOUR_SECRET
          parameter:
            source: header
            name: X-Hub-Signature-256
      - match:
          type: value
          value: refs/heads/main
          parameter:
            source: payload
            name: ref
```

After editing `hooks.yaml`: `systemctl restart webhook`

### Adding a New Project

**For private repos** — SSH deploy key needed for `git pull`:
```bash
ssh-keygen -t ed25519 -C "100-ubuntu / project-name" -f ~/.ssh/id_ed25519_project-name -N ''
cat ~/.ssh/id_ed25519_project-name.pub  # paste into GitHub → repo Settings → Deploy keys (read-only)
```
Add host alias to `~/.ssh/config`:
```
Host github-project-name
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_project-name
  IdentitiesOnly yes
```
Clone using the alias:
```bash
git clone git@github-project-name:caduff97/project-name.git ~/containers/project-name
```

**For public repos** — clone directly:
```bash
git clone https://github.com/caduff97/project-name.git ~/containers/project-name
```

**Then for all projects:**

1. Generate a webhook secret:
   ```bash
   openssl rand -hex 32
   ```

2. Add an entry to `/root/webhook/hooks.yaml` (copy the structure above, update id/secret/branch)

3. Restart the webhook service:
   ```bash
   systemctl restart webhook
   ```

4. Add a GitHub webhook on the repo → Settings → Webhooks:
   - Payload URL: `https://webhooks.carlostri.com/hooks/project-name`
   - Content type: `application/json`
   - Secret: the generated secret
   - Events: push only

5. If public via Cloudflare Tunnel, add its Docker network to `~/containers/cloudflared/docker-compose.yml` and redeploy cloudflared

6. Push to `main` — first deploy triggers automatically

### Security Notes

- The webhook service runs as root — this is effectively required for Docker operations
- HMAC signature validation on every request prevents unauthorized deploys
- Protect the `main` branch on GitHub (require PR + 2FA) to prevent unauthorized pushes triggering deploys

## Container Orchestration

### Dockge

Stack management UI that discovers and controls compose files on disk. The folder structure is the source of truth — Dockge is just a UI layer on top of it.

Access at `https://dockge.internal`

```yaml
services:
  dockge:
    image: louislam/dockge:1
    restart: unless-stopped
    ports:
      - "127.0.0.1:5001:5001"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data
      - /root/containers:/opt/stacks
    environment:
      - DOCKGE_STACKS_DIR=/opt/stacks
```

### Stack Organization

Each service lives in its own folder under `~/containers/`:

```
~/containers/
├── caddy/          ← reverse proxy + local CA
├── cloudflared/    ← Cloudflare Tunnel
├── dnsmasq/        ← *.internal DNS resolution
├── dockge/         ← stack management UI
├── uptime-kuma/    ← monitoring
├── dailytxt/       ← journal app
├── capitallog/     ← investments manager
├── bnbturf/        ← public app
├── bonafideready/  ← public app
├── carlostri/      ← public app
├── scorebuddies/   ← public app
├── nextcloud/      ← data in /tank/nextcloud
└── immich/         ← data in /tank/immich
```

Each folder contains a `docker-compose.yml`, optional `.env`, and a `data/` directory for persistent volumes. Large-data services (Nextcloud, Immich) reference `/mnt/data/<service>` via `.env` instead of `./data/`.

## Nextcloud

Stack: postgres + redis + nextcloud + cron, all on an internal `backend` bridge network. The `cron` service uses the same nextcloud image with `entrypoint: /cron.sh` — runs background jobs every 5 minutes, no host crontab needed.

Access at `https://nextcloud.internal`

### Required occ Commands

Run after first start or migration:

```bash
docker exec -u www-data nextcloud-nextcloud-1 php occ config:system:set trusted_domains 0 --value="nextcloud.internal"
docker exec -u www-data nextcloud-nextcloud-1 php occ config:system:set overwrite.cli.url --value="https://nextcloud.internal"
docker exec -u www-data nextcloud-nextcloud-1 php occ config:system:set overwriteprotocol --value="https"
docker exec -u www-data nextcloud-nextcloud-1 php occ config:system:set trusted_proxies 0 --value="127.0.0.1"
docker exec -u www-data nextcloud-nextcloud-1 php occ background:cron
docker exec -u www-data nextcloud-nextcloud-1 php occ db:add-missing-indices
```

### Caddy Entry

HSTS header required to avoid browser warnings:

```
nextcloud.internal {
  reverse_proxy 127.0.0.1:8080
  header Strict-Transport-Security "max-age=15552000"
}
```

### Migration Gotchas

- `dbhost` in config.php must match the compose service name (`postgres`, not the old container name)
- `dbuser` in config.php is usually `oc_<suffix>` from the original install — update it to your `POSTGRES_USER`
- `dbpassword` must match the actual postgres user password (set via `ALTER USER ... WITH PASSWORD`)
- Stop Nextcloud before restoring a dump — if it starts first, it initializes a fresh schema and the restore will conflict with duplicate primary keys

## Monitoring

### Uptime Kuma

Monitoring dashboard with alerting. Mounts the Caddy root CA so it can verify `*.internal` services over HTTPS.

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    restart: unless-stopped
    ports:
      - "127.0.0.1:3001:3001"
    volumes:
      - ./data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /root/containers/caddy/data/caddy/pki/authorities/local/root.crt:/usr/local/share/ca-certificates/caddy-root.crt:ro
    environment:
      - NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/caddy-root.crt
```

Access at `https://uptime.internal`

## Common Operations

### Restarting the Tunnel

```bash
docker restart cloudflared
docker logs cloudflared --tail 20  # Verify it reconnected
```

### Checking What's Running

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### Debugging Network Issues

```bash
# Test DNS resolution
nslookup some-domain.com

# Test Tailscale connectivity
tailscale ping other-device

# Check what ports are listening
ss -tlnp
```

