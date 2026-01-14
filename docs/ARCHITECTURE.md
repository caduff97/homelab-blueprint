# Architecture Guide

This document covers the technical implementation details. For the overview and rationale, see the [README](../README.md).

## Network Interfaces

| Interface | Purpose |
|-----------|---------|
| `eno1` | LAN - shared network (roommates use this too) |
| `tailscale0` | VPN - my personal devices only |
| `docker0` | Docker bridge network |

The server sits on a shared LAN but private services are only accessible via Tailscale. Roommates can't accidentally (or intentionally) access my stuff.

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
2. Add the network to cloudflared's compose file
3. Redeploy cloudflared: `docker compose up -d`
4. In Cloudflare Zero Trust dashboard:
   - Go to Networks → Tunnels → my tunnel → Public Hostname
   - Add hostname: `myapp.mydomain.com`
   - Service: `http://myapp-container:80`
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

Don't forget UFW rules to allow Docker networks to reach the host:
```bash
sudo ufw allow from 172.16.0.0/12 to any port PORT_NUMBER
```

## Tailscale Serve Setup

### How It Works

Tailscale Serve acts as a reverse proxy that:
1. Listens on my Tailscale IP (100.x.x.x)
2. Terminates TLS with a valid certificate
3. Forwards to localhost services

This is simpler than running Caddy/nginx because Tailscale handles certs automatically via its ACME integration.

### Adding a Private Service

1. Configure the container to bind to localhost only:
   ```yaml
   ports:
     - "127.0.0.1:8080:80"
   ```

2. Add a Tailscale serve rule:
   ```bash
   tailscale serve --bg --https=8080 http://127.0.0.1:8080
   ```

3. Access at `https://hostname.tailnet.ts.net:8080`

### Managing Serve Rules

```bash
# View current configuration
tailscale serve status

# Remove a rule
tailscale serve --https=8080 off

# The --bg flag runs it persistently (survives reboots)
```

### Why Localhost Binding Matters

By binding to `127.0.0.1` instead of `0.0.0.0`, the service is unreachable from the LAN. Even if someone on my network scans for open ports, they won't find my private services. Only Tailscale Serve (running on the same machine) can reach them.

## DNS Architecture

### The Flow

```
My phone/laptop (on Tailscale)
        │
        ▼
Tailscale DNS (100.100.100.100)
        │
        ├──► *.tailnet.ts.net → MagicDNS (resolves Tailscale hostnames)
        │
        └──► Everything else
                │
                ▼
          Pi-Hole (server's Tailscale IP)
                │
                ├──► Blocked domains → 0.0.0.0
                │
                └──► Allowed → Upstream DNS
```

### Configuration

1. **On the server**: Tailscale manages `/etc/resolv.conf` (set `accept-dns=true`)
2. **In Tailscale admin**: Set Pi-Hole's Tailscale IP as the global nameserver
3. **In Pi-Hole**: Use `network_mode: host` and bind web UI to localhost

```yaml
# Pi-Hole compose
services:
  pihole:
    image: pihole/pihole:latest
    network_mode: host
    environment:
      FTLCONF_webserver_port: '127.0.0.1:1010'  # Web UI on localhost only
      FTLCONF_dns_listeningMode: 'all'          # Accept DNS from all interfaces
```

This gives me ad-blocking on every device connected to my Tailnet, anywhere in the world.

## Storage Layout

| Mount | Purpose | Size |
|-------|---------|------|
| `/` | OS, Docker engine | ~120GB |
| `/home` | User data, container volumes | ~100GB |
| `/mnt/data` | Large files (photos, cloud storage) | ~2TB |
| `/mnt/backup` | Restic repository | ~120GB |

### Container Data Strategy

I keep all persistent container data under `~/containers/`:
```
containers/
├── immich_postgres/
├── nextcloud_postgres/
├── app1_data/
└── app2_data/
```

This makes backups simple - one directory to include in Restic.

## Firewall (UFW)

### Current Rules

```bash
# Pi-Hole DNS (LAN devices need this)
sudo ufw allow 53

# Docker-to-host (for non-containerized services via cloudflared)
sudo ufw allow from 172.16.0.0/12 to any port SPECIFIC_PORTS
```

### What's NOT Exposed

- No 80/443 - Cloudflare Tunnel handles this
- No VPN ports - Tailscale uses NAT traversal
- Private service ports - bound to localhost, only Tailscale Serve reaches them

## Container Orchestration

### Portainer

I use Portainer for day-to-day container management. It's accessed via Tailscale Serve on port 9443.

Note: Portainer uses a self-signed cert internally. Configure Tailscale Serve with:
```bash
tailscale serve --bg --https=9443 https+insecure://127.0.0.1:9443
```

### Stack Organization

Each application gets its own stack (compose file) in Portainer. This keeps networks isolated - apps can't talk to each other unless explicitly connected.

## Monitoring

### Uptime Kuma

Runs in `network_mode: host` to monitor both localhost services and external endpoints. Sends alerts via email when something goes down.

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    network_mode: host
    volumes:
      - uptime-kuma-data:/app/data
```

Access via Tailscale Serve on port 3001.

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

### After Container Restarts

If I see 502 errors after restarting containers, nginx inside the frontend might have cached stale DNS for the backend. Restarting the frontend container fixes it.
