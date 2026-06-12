# ☁️ Proxmox + Nextcloud AIO with External NTFS Hard Drive

> Self-hosted personal cloud storage using **Nextcloud All-in-One (AIO)** inside a **Privileged LXC container** on Proxmox VE, with external NTFS drives mounted as storage — accessible from anywhere via Tailscale.

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Hardware & Environment](#hardware--environment)
3. [Why LXC Instead of VM](#why-lxc-instead-of-vm)
4. [Step-by-Step Setup](#step-by-step-setup)
   - [Step 1 — Download Ubuntu Template](#step-1--download-ubuntu-template)
   - [Step 2 — Create the LXC Container](#step-2--create-the-lxc-container)
   - [Step 3 — Configure Bind Mounts & Nesting](#step-3--configure-bind-mounts--nesting)
   - [Step 4 — Install Docker](#step-4--install-docker)
   - [Step 5 — Run Nextcloud AIO](#step-5--run-nextcloud-aio)
   - [Step 6 — Set Up Remote Access via Tailscale](#step-6--set-up-remote-access-via-tailscale)
   - [Step 7 — Configure Nextcloud Domain & Start Containers](#step-7--configure-nextcloud-domain--start-containers)
   - [Step 8 — Add External Storage Mounts in Nextcloud](#step-8--add-external-storage-mounts-in-nextcloud)
5. [Proxmox Remote Access via Tailscale](#proxmox-remote-access-via-tailscale)
6. [Troubleshooting](#troubleshooting)
7. [Architecture Diagram](#architecture-diagram)

---

## Overview

This project replaces Google Drive with a fully self-hosted alternative. Nextcloud AIO is deployed inside a **Privileged LXC container** on Proxmox, and three external NTFS partitions (`/mnt/Mobile`, `/mnt/Computer`, `/mnt/Others`) are passed through via **bind mounts** — preserving all existing data without reformatting.

**Key goals:**
- Keep all existing data on the NTFS drives intact
- Make files accessible from outside the home
- Use free tools only (no paid domain or cloud services)

---

## Hardware & Environment

| Component | Detail |
|-----------|--------|
| **Hypervisor** | Proxmox VE 9.2.3 |
| **Internal Disk** | `/dev/sda` — ~232 GB |
| **External HDD** | `/dev/sdb` — ~596 GB NTFS |
| **Partition 1** | `/dev/sdb1` → `/mnt/Mobile` |
| **Partition 2** | `/dev/sdb2` → `/mnt/Computer` |
| **Partition 3** | `/dev/sdb3` → `/mnt/Others` |
| **RAM** | 8 GB total |
| **Container OS** | Ubuntu 24.04 LTS |
| **Container ID** | 1991 |

---

## Why LXC Instead of VM

An earlier attempt used an Ubuntu VM with disk pass-through, but this failed because the external drive was already mounted on the Proxmox host — making raw pass-through incompatible.

**LXC with bind mounts** was the correct approach:
- Lighter on resources than a full VM
- Direct access to host-mounted NTFS partitions via `mp` (mount point) directives
- No disk reformatting needed
- Works seamlessly with existing data

---

## Step-by-Step Setup

### Step 1 — Download Ubuntu Template

In the Proxmox web UI:

1. Click **local** (your node's storage) in the left sidebar
2. Click **CT Templates** → **Templates** button
3. Search for `ubuntu-24.04-standard` and click **Download**
4. Wait for `TASK OK`

---

### Step 2 — Create the LXC Container

Click **Create CT** in the top-right corner and fill in:

| Tab | Setting | Value |
|-----|---------|-------|
| **General** | CT ID | `1991` (auto-assigned) |
| | Hostname | `nextcloud` |
| | Unprivileged container | ❌ **Unchecked** (must be Privileged) |
| | Password | Choose a strong root password |
| **Template** | Template | `ubuntu-24.04-standard` |
| **Disks** | Disk size | `20 GB` (OS + Docker only; data lives on external drives) |
| **CPU** | Cores | `2–4` |
| **Memory** | RAM | `4096 MB` (4 GB) |
| | Swap | `512 MB` |
| **Network** | IPv4 | Static — e.g. `192.168.1.150/24` |
| | Gateway | Your router IP (e.g. `192.168.1.1`) |

> ⚠️ **Important:** When creating a Privileged container, the Nesting option is greyed out in the UI. Do NOT start the container yet — we will enable Nesting manually in the next step.

Click **Finish** and wait for `TASK OK`.

---

### Step 3 — Configure Bind Mounts & Nesting

Open the **Proxmox Node Shell** (not the container) and run:

```bash
nano /etc/pve/lxc/1991.conf
```

Scroll to the bottom and add these lines:

```ini
features: nesting=1
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
mp0: /mnt/Mobile,mp=/mnt/Mobile
mp1: /mnt/Computer,mp=/mnt/Computer
mp2: /mnt/Others,mp=/mnt/Others
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`.

> **What this does:**
> - `nesting=1` — allows Docker to run inside the LXC
> - `lxc.apparmor.profile: unconfined` — fixes the AppArmor error that blocks Docker on Ubuntu 24.04 inside privileged LXC
> - `mp0/mp1/mp2` — mirrors the three NTFS partitions into the container at the same paths

Now start the container from the Proxmox UI and open its **Console**.

Verify the drives are accessible:

```bash
ls -la /mnt/Mobile /mnt/Computer /mnt/Others
```

You should see your existing files listed.

---

### Step 4 — Install Docker

Inside the container console (logged in as root):

```bash
apt update && apt upgrade -y
apt install curl wget git -y

curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

systemctl status docker
# Should show: active (running)
```

---

### Step 5 — Run Nextcloud AIO

Run this Docker command inside the container. The key flags:
- `APACHE_PORT=11000` — required for Reverse Proxy / Tunnel mode
- `SKIP_DOMAIN_VALIDATION=true` — lets you enter the domain manually without port 443 being open
- `/mnt:/mnt:rw` — exposes all three NTFS partitions to Nextcloud's Docker containers

```bash
sudo docker run \
  --init \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 8080:8080 \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  --volume /mnt:/mnt:rw \
  -e APACHE_PORT=11000 \
  -e SKIP_DOMAIN_VALIDATION=true \
  nextcloud/all-in-one:latest
```

> ⚠️ **Note:** Do NOT publish ports 80 or 443 here. Publishing them caused a conflict with Nextcloud AIO's internal domain checker, which would block the setup page.

After the container starts, retrieve the setup password:

```bash
sudo docker logs nextcloud-aio-mastercontainer | grep -A2 "complete"
# Or once you've visited the web UI once:
sudo docker exec nextcloud-aio-mastercontainer \
  cat /mnt/docker-aio-config/data/configuration.json | \
  grep -oP '"password":\s*"\K[^"]+'
```

---

### Step 6 — Set Up Remote Access via Tailscale

Since a free domain that works with Cloudflare Tunnels isn't straightforward, **Tailscale** was used instead — it provides a free, always-on VPN with automatic HTTPS certificates.

Inside the container:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

If `tailscale up` fails with a TUN device error:

**On the Proxmox host**, add to `/etc/pve/lxc/1991.conf`:

```ini
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Reboot the container, then inside the container:

```bash
systemctl enable tailscaled
systemctl start tailscaled
tailscale up
```

Follow the printed URL to authenticate with your Google/GitHub account.

After login, get your Tailscale domain:

```bash
tailscale status --json | grep -i "fqdn"
# Example output: "nextcloud.<your-tailscale>.ts.net"
```

Issue a certificate:

```bash
tailscale cert nextcloud.<your-tailscale>.ts.net
```

---

### Step 7 — Configure Nextcloud Domain & Start Containers

Open your browser and navigate to:

```
https://<LXC_IP>:8080
```

Accept the self-signed certificate warning. Enter the setup password you retrieved earlier.

In the **Domain** field, enter your Tailscale domain:

```
nextcloud.<your-tailscale>.ts.net
```

Click **Submit**. Nextcloud AIO will now download and start all service containers (database, Redis, Nextcloud app, etc.). This takes 5–10 minutes.

When complete, your admin credentials will be displayed. Save them.

---

### Step 8 — Add External Storage Mounts in Nextcloud

Once logged into Nextcloud as admin:

1. Click your avatar (top-right) → **Apps**
2. Under **Files**, find and enable **External storage support**
3. Go to **Administration Settings** → **External storages**
4. Add three entries:

| Folder Name | Storage Type | Configuration |
|-------------|-------------|---------------|
| `Mobile` | Local | `/mnt/Mobile` |
| `Computer` | Local | `/mnt/Computer` |
| `Others` | Local | `/mnt/Others` |

Each should show a **green dot** when the path is accessible.

Your three NTFS drives now appear as folders in the Nextcloud Files app, readable and writable from any device via the internet.

---

## Proxmox Remote Access via Tailscale

To access the **Proxmox web UI** securely from outside the home:

On the **Proxmox host** shell:

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --hostname=proxmox

# Issue a certificate for the Proxmox domain
tailscale cert proxmox.<your-tailscale>.ts.net

# Install it into Proxmox
pvenode cert set proxmox.<your-tailscale>.ts.net.crt proxmox.<your-tailscale>.ts.net.key \
  --force 1 --restart 1

# Forward port 443 → 8006 so you don't need to type the port
tailscale serve --bg tcp:443 tcp://127.0.0.1:8006
```

Now access Proxmox at:

```
https://proxmox.<your-tailscale>.ts.net
```

No port number needed, no browser security warnings, and a Let's Encrypt certificate issued automatically by Tailscale.

---

## Troubleshooting

### ❌ AppArmor error when starting Docker

**Error:**
```
apparmor_parser: Unable to replace "docker-default". Access denied.
```

**Fix:** Add to `/etc/pve/lxc/1991.conf` on the Proxmox host:

```ini
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop:
```

Reboot the container.

---

### ❌ Nextcloud AIO shows "Domaincheck container is not running"

**Cause:** Ports 80 and 443 are published, so AIO tries to auto-verify the domain over the internet — which fails in a local/tunnel setup.

**Fix:** Remove `--publish 80:80` and `--publish 443:443` from the Docker run command. Use only `--publish 8080:8080`. Also add `-e APACHE_PORT=11000 -e SKIP_DOMAIN_VALIDATION=true`.

Then reset and restart:

```bash
sudo docker stop nextcloud-aio-mastercontainer
sudo docker rm nextcloud-aio-mastercontainer
sudo docker volume rm nextcloud_aio_mastercontainer
# Re-run the corrected docker run command
```

---

### ❌ Setup password not visible in logs

**Fix:** Open `https://<LXC_IP>:8080` in your browser first (this triggers password file creation), then run:

```bash
sudo docker exec nextcloud-aio-mastercontainer \
  cat /mnt/docker-aio-config/data/configuration.json | \
  grep -oP '"password":\s*"\K[^"]+'
```

---

### ❌ Tailscale `tailscale up` fails — "doesn't appear to be running"

**Fix:** Inside the container:

```bash
systemctl enable tailscaled
systemctl start tailscaled
tailscale up
```

If it still fails, add TUN device access in `/etc/pve/lxc/1991.conf`:

```ini
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Reboot the container.

---

### ❌ Proxmox gives HTTP 502 after `tailscale serve`

**Cause:** `tailscale serve https://127.0.0.1:8006` fails because Proxmox uses a self-signed cert internally, which Tailscale rejects.

**Fix:** Use `tcp` mode instead of `https` mode:

```bash
tailscale serve reset
tailscale cert proxmox.<your-tailscale>.ts.net
pvenode cert set proxmox.<your-tailscale>.ts.net.crt proxmox.<your-tailscale>.ts.net.key \
  --force 1 --restart 1
tailscale serve --bg tcp:443 tcp://127.0.0.1:8006
```

---

## Architecture Diagram

```
Internet / Phone (outside home)
         │
         │  Tailscale encrypted tunnel
         ▼
┌─────────────────────────────────────┐
│         Proxmox VE 9.2.3            │
│  ┌─────────────────────────────┐    │
│  │   LXC 1991 (Ubuntu 24.04)   │    │
│  │   ┌──────────────────────┐  │    │
│  │   │  Docker              │  │    │
│  │   │  nextcloud-aio-...   │  │    │
│  │   └──────────────────────┘  │    │
│  │   Bind Mounts:              │    │
│  │   /mnt/Mobile   ─────────── │─── │── /dev/sdb1 (NTFS)
│  │   /mnt/Computer ─────────── │─── │── /dev/sdb2 (NTFS)
│  │   /mnt/Others   ─────────── │─── │── /dev/sdb3 (NTFS)
│  └─────────────────────────────┘    │
│  Tailscale: <your-tailscale>.ts.net │
└─────────────────────────────────────┘
         │
         ▼
   Home Router (Vodafone Station)
         │
       LAN devices
```
