# 🏠 Homelab Projects on Proxmox VE

A collection of self-hosted services built on top of **Proxmox VE 9.2.3**, running on a home server with a 250 GB internal SSD and external NTFS hard drives.

---

## 📁 Projects

| # | Project | Description |
|---|---------|-------------|
| 01 | [Proxmox + Nextcloud](./01-proxmox-nextcloud/) | Self-hosted cloud storage via Nextcloud AIO inside a Privileged LXC, using external NTFS drives for data |
| 02 | [Proxmox + Pi-hole](./02-proxmox-pihole/) | Network-wide DNS ad-blocker running in a dedicated Debian LXC container |
| 03 | [Proxmox + Media Server](./03-proxmox-mediaserver/) | Jellyfin media server in LXC, scanning external drives and streaming to Sony TV via DLNA |

---

## 🖥️ Host Environment

| Component | Detail |
|-----------|--------|
| **Hypervisor** | Proxmox VE 9.2.3 |
| **Internal Disk** | `/dev/sda` — ~232 GB (OS + LVM) |
| **External HDD** | `/dev/sdb` — ~596 GB, split into 3 NTFS partitions |
| **Partitions** | `sdb1 → /mnt/Mobile`, `sdb2 → /mnt/Computer`, `sdb3 → /mnt/Others` |
| **RAM** | 8 GB |
| **Remote Access** | Tailscale VPN (`<your-tailscale>.ts.net` network) |

---

## 🔐 Remote Access

All three services are accessible securely from outside the home via **Tailscale**, eliminating the need for port-forwarding:

- **Proxmox UI:** `https://proxmox.<your-tailscale>.ts.net` (no port needed — served via `tailscale serve tcp:443`)
- **Nextcloud:** `https://nextcloud.<your-tailscale>.ts.net` (via Tailscale MagicDNS + HTTPS cert)
- **Pi-hole:** accessible inside the Tailscale network

---

## 📌 Notes

- All LXC containers use **Privileged** mode to handle NTFS bind-mounts without permission issues.
- `Nesting=1` is enabled manually in `/etc/pve/lxc/<id>.conf` for containers running Docker or systemd services.
- AppArmor restrictions are lifted in the Nextcloud container with `lxc.apparmor.profile: unconfined`.
