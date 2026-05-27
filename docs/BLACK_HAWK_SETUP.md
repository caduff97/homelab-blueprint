# black-hawk Setup Plan

Repurposing the old Dell desktop into a second Proxmox node — development machine, Proxmox Backup Server, and print server.

## Goals

| Role | How |
|------|-----|
| Daily workstation | Ubuntu Desktop VM with direct display output (keyboard, 2 monitors) |
| Backup server | Proxmox Backup Server (PBS) — receives vzdump + Restic from HP EliteDesk |
| Print server | Dedicated LXC — USB printer shared over the network |
| Dev environments | Extra LXCs/VMs spun up as needed, isolated per project |

## Hardware

| Drive | Size | Planned Role |
|-------|------|-------------|
| 240GB SSD | 240GB | Proxmox OS + VM/LXC local disks |
| 500GB SSD | 500GB | Additional VM storage or secondary PBS datastore |
| 1.8TB HDD | 1.8TB | **Primary PBS datastore** |

## Network

Will be on Wi-Fi (no LAN cable in the room). Proxmox doesn't support Wi-Fi bridging natively, so LXCs/VMs use NAT through the host's `wlan0`.

**Setup order**: install and configure everything on LAN first, then migrate to Wi-Fi.

### NAT Networking Layout

```
Wi-Fi router
     │
  wlan0 (Proxmox host — home LAN IP)
     │
  vmbr0 (internal bridge — 10.10.10.1/24)
     │
  LXCs / VMs (10.10.10.x — NAT'd through wlan0)
```

Access from other devices: via Tailscale (already on all personal devices). No need for LAN-direct access.

## Display Output (Workstation VM)

The i5-6500 (Skylake) supports **Intel GVT-g** — GPU virtualization that lets a VM share the integrated GPU (HD Graphics 530) and output to physical monitors.

The Ubuntu Desktop VM will:
- Use a GVT-g vGPU for display output to the motherboard's ports
- Have keyboard + mouse passed through via USB passthrough
- Auto-start on Proxmox boot

### Enabling GVT-g on Proxmox

```bash
# 1. Enable IOMMU in GRUB (/etc/default/grub)
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt i915.enable_gvt=1"
update-grub

# 2. Load required kernel modules (/etc/modules)
kvmgt
vfio-iommu-type1
vfio-mdev

# 3. Reboot, verify
dmesg | grep -i gvt
ls /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/
```

### Creating the vGPU

```bash
# Find available GVT-g types (e.g. i915-GVTg_V5_4)
ls /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/

# Create a vGPU instance (use a random UUID)
UUID=$(uuidgen)
echo $UUID > /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/i915-GVTg_V5_4/create
```

Then in the VM config (via Proxmox UI or `/etc/pve/qemu-server/<vmid>.conf`):
```
hostpci0: 0000:00:02.0,mdev=<UUID>
vga: none
```

## Planned LXC/VM Layout

| Container | Type | Purpose |
|-----------|------|---------|
| Ubuntu Desktop | VM | Daily workstation — display via GVT-g, USB passthrough |
| PBS | LXC or native | Proxmox Backup Server |
| Print server | LXC | CUPS — USB printer shared over network |
| Dev envs | LXC (as needed) | Isolated environments per project |

## Installation Order

1. **Install Proxmox** on the 240GB SSD (boot from USB)
2. **Configure Wi-Fi** — `wlan0` + NAT bridge (`vmbr0`) in `/etc/network/interfaces`
3. **Attach 1.8TB HDD** — format + add as Proxmox Directory storage for PBS
4. **Install PBS** — either as a dedicated LXC or natively on the host
5. **Configure PBS** on HP EliteDesk — point backup jobs at black-hawk
6. **Enable GVT-g** — follow steps above, reboot
7. **Create Ubuntu Desktop VM** — assign vGPU, USB passthrough, auto-start
8. **Create print server LXC** — install CUPS, share printer
9. **Switch from LAN to Wi-Fi** — update `/etc/network/interfaces`, reboot

## Wi-Fi Configuration

`/etc/network/interfaces` on the Proxmox host:

```
auto lo
iface lo inet loopback

# Wi-Fi (host uplink)
auto wlan0
iface wlan0 inet dhcp
    wpa-ssid "YourSSID"
    wpa-psk "YourPassword"

# Internal NAT bridge for VMs/LXCs
auto vmbr0
iface vmbr0 inet static
    address 10.10.10.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    post-up echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o wlan0 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s 10.10.10.0/24 -o wlan0 -j MASQUERADE
```

## Gotchas

- Set up and test everything on LAN before switching to Wi-Fi
- GVT-g initial setup requires 1-2 hours and a few reboots — do it before creating VMs
- PBS requires the `proxmox-backup-server` package — can run on plain Debian/Ubuntu, doesn't need a full Proxmox install, but running it on the Proxmox host keeps management in one place
- USB passthrough for keyboard/mouse: add devices to VM config via Proxmox UI (Hardware → Add → USB Device) — use "Use USB vendor/device ID" for reliable mapping
