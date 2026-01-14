# Lessons Learned

Principles I've learned from building and maintaining this homelab. These aren't technical how-tos - they're the philosophy behind decisions that lead to a reliable, low-stress setup.

## Backup What Matters

Data falls into two categories: replaceable and irreplaceable.

Replaceable data (configs, containers, apps) can be rebuilt with time and effort. Irreplaceable data (photos, personal documents, databases with years of history) cannot.

**Principle**: Identify what's irreplaceable and protect it first. Everything else can wait.

I started with database backups because losing years of app data would hurt. The photo library backup came later - still working on it. But I know what matters and I'm protecting it in order of importance.

Don't wait for the perfect backup solution. Start with something that covers the critical stuff today.

## Document Everything

Memory is unreliable. Six months from now, I won't remember why I configured something a certain way, what port conflicts I solved, or which workaround fixed that weird issue.

**Principle**: Write it down while it's fresh. Future me will thank present me.

This documentation exists because I got tired of re-solving the same problems. Now when something breaks, I check the docs first. Usually past me already figured it out.

Documentation also forces clarity. If I can't explain a setup in writing, I probably don't understand it well enough.

## Small Commits, Clear History

Git isn't just for code. Infrastructure configs, scripts, documentation - all of it benefits from version control.

**Principle**: Commit often, with meaningful messages. Each commit should be one logical change.

Small commits make rollbacks surgical. If something breaks after a change, I can revert just that change, not a week's worth of modifications bundled together.

Clear commit messages are notes to future me: "Why did I change this?" The message should answer that question.

## Closed Ports, Open Mind

Every open port is an invitation. Even with firewalls and security updates, exposed services are attack surface.

**Principle**: Don't open ports. Use tunnels and VPNs instead.

Cloudflare Tunnel handles public traffic without exposing my IP or opening router ports. Tailscale handles private access with device-level authentication. My router's port forwarding page is empty.

This isn't paranoia - it's simplicity. No ports to manage, no dynamic DNS to maintain, no SSL certificates to renew manually. The "secure" approach turned out to be the "easy" approach.

## Peace of Mind Through Visibility

The worst feeling is not knowing if something is broken. Is the backup running? Is that service up? Did the SSL cert renew?

**Principle**: Automate monitoring and notifications. If something fails, I should know before users do.

Uptime Kuma watches my services and emails me when something goes down. Backup scripts send completion notifications. I check dashboards occasionally, but mostly I trust that silence means everything is fine.

This transforms anxiety into confidence. I don't wonder if things are working - I know they are, or I get alerted when they're not.

## Simplicity Over Cleverness

Complex setups are fragile setups. Every extra layer, every clever workaround, every "I'll remember how this works" is future debt.

**Principle**: Choose boring solutions. They're easier to debug at 2 AM.

I removed my reverse proxy (Caddy) because Tailscale Serve does the same thing with less configuration. I use Portainer's UI instead of managing compose files across directories. I pick the approach with fewer moving parts.

When something breaks in a simple system, there are fewer places to look.

## Separation of Concerns

Public services and private services have different needs. Mixing them creates compromise.

**Principle**: Split traffic by trust level. Public goes one path, private goes another.

Public services route through Cloudflare - they get DDoS protection, caching, and my IP stays hidden. Private services route through Tailscale - they get device authentication and never touch the public internet.

Neither path knows about the other. A compromise in one doesn't affect the other.

## Defense in Depth

Security isn't one lock - it's layers. If one layer fails, others should still protect.

**Principle**: Assume each layer might be bypassed. Add another layer anyway.

My private services bind to localhost only. Even if someone gets on my LAN, they can't reach those ports. Tailscale Serve is the only path in.

Is this overkill? Maybe. But it costs nothing and means I don't have to trust any single mechanism completely.

## Start Imperfect, Iterate

Waiting for the perfect setup means never starting. Real systems evolve.

**Principle**: Get something working, then improve it. Progress beats perfection.

My backup strategy started as a manual script I ran occasionally. Then I added cron. Then email notifications. Then retention policies. Each step was an improvement over the last.

The photo backup still isn't solved. But everything else is protected while I figure that out.

## Own Your Data

Cloud services are convenient until they're not. Pricing changes, features disappear, companies shut down.

**Principle**: Self-host what matters. Accept the responsibility that comes with it.

I run Immich instead of Google Photos. Nextcloud instead of Dropbox. Pi-Hole instead of hoping my router's DNS is private.

This means I'm responsible for backups, updates, and uptime. That's the trade-off. But my data stays mine, and no algorithm decides what I see.

---

These principles took time to learn, usually through mistakes. The technical details in the other docs will change as tools evolve. These ideas won't.
