# 🎬 Proxmox + Jellyfin Home Media Server

> **Jellyfin** deployed in a dedicated **Privileged LXC container** on Proxmox VE, scanning three external NTFS drives and streaming photos and videos to a Sony Smart TV via DLNA — fully controllable with the TV remote, no phone needed.

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Why Jellyfin](#why-jellyfin)
3. [Hardware & Environment](#hardware--environment)
4. [Step-by-Step Setup](#step-by-step-setup)
   - [Step 1 — Create the LXC Container](#step-1--create-the-lxc-container)
   - [Step 2 — Pass External Drives to the Container](#step-2--pass-external-drives-to-the-container)
   - [Step 3 — Fix Network / DNS Issues](#step-3--fix-network--dns-issues)
   - [Step 4 — Install Jellyfin](#step-4--install-jellyfin)
   - [Step 5 — Initial Jellyfin Configuration](#step-5--initial-jellyfin-configuration)
   - [Step 6 — Add Media Libraries (Smart Filtering)](#step-6--add-media-libraries-smart-filtering)
   - [Step 7 — Enable DLNA for the TV](#step-7--enable-dlna-for-the-tv)
5. [Watching on Sony TV with Remote Only](#watching-on-sony-tv-with-remote-only)
6. [Troubleshooting](#troubleshooting)
7. [Architecture Diagram](#architecture-diagram)

---

## Overview

This project turns a home server into a **personal media center** — similar to Netflix, but using your own files stored on external drives. The goal was to browse family photos and videos on a Sony Smart TV using just the TV remote, with no phone or laptop involvement.

Jellyfin scans the three NTFS partitions, filters to show **only photos and videos** (hiding all other file types), generates thumbnails, and serves them via DLNA — which the Sony TV discovers automatically on the local network.

---

## Why Jellyfin

| Feature | Jellyfin | Plex | MiniDLNA |
|---------|----------|------|----------|
| **Cost** | Free & open-source | Free + paid tiers | Free |
| **Media filtering** | ✅ Smart (by type) | ✅ | ❌ |
| **Thumbnail generation** | ✅ Automatic | ✅ | ❌ |
| **DLNA support** | ✅ Built-in | ✅ (with PlexAmp) | ✅ |
| **Web UI** | ✅ | ✅ | ❌ |
| **Phone account required** | ❌ | ✅ | ❌ |

Jellyfin was chosen because it's completely free, generates thumbnails automatically, and has built-in DLNA without requiring a Plex account.

---

## Hardware & Environment

| Component | Detail |
|-----------|--------|
| **Hypervisor** | Proxmox VE 9.2.3 |
| **Container ID** | 102 |
| **Container OS** | Debian 12 |
| **Container type** | **Privileged** (required for NTFS bind mounts) |
| **External drives** | `/mnt/Mobile`, `/mnt/Computer`, `/mnt/Others` (mounted on host) |
| **TV** | Sony Smart TV (DLNA compatible) |
| **Jellyfin port** | `8096` (web UI) |

---

## Step-by-Step Setup

### Step 1 — Create the LXC Container

In the Proxmox web UI, click **Create CT**:

| Tab | Setting | Value |
|-----|---------|-------|
| **General** | CT ID | `102` |
| | Hostname | `jellyfin-server` |
| | Unprivileged container | ❌ **Unchecked** (must be Privileged for NTFS) |
| | Password | Choose a strong root password |
| **Template** | Template | `debian-12-standard` |
| **Disks** | Size | `16 GB` (thumbnails and metadata only; media stays on drives) |
| **CPU** | Cores | `2` |
| **Memory** | RAM | `2048 MB` (needed for thumbnail generation) |
| **Network** | IPv4 | **DHCP** or Static (e.g. `192.168.1.102/24`) |
| | Gateway | Your router IP |

Click **Finish** but **do not start yet**.

---

### Step 2 — Pass External Drives to the Container

The external NTFS drives are already mounted on the **Proxmox host** at `/mnt/Mobile`, `/mnt/Computer`, and `/mnt/Others`. We pass them into the Jellyfin container using bind mounts.

On the **Proxmox Node Shell**:

```bash
pct set 102 -mp0 /mnt/Mobile,mp=/mnt/Mobile
pct set 102 -mp1 /mnt/Computer,mp=/mnt/Computer
pct set 102 -mp2 /mnt/Others,mp=/mnt/Others
```

Verify by checking `/etc/pve/lxc/102.conf`:

```bash
cat /etc/pve/lxc/102.conf
```

You should see:

```ini
mp0: /mnt/Mobile,mp=/mnt/Mobile
mp1: /mnt/Computer,mp=/mnt/Computer
mp2: /mnt/Others,mp=/mnt/Others
```

Now start the container from the Proxmox UI.

---

### Step 3 — Fix Network / DNS Issues

Open the container **Console** and log in as root.

If Pi-hole is running on your network, the container may use it as DNS — which can fail during installation if Pi-hole is unreachable. Fix:

```bash
nano /etc/resolv.conf
```

Add at the top:

```
nameserver 8.8.8.8
nameserver 8.8.4.4
```

Test:

```bash
apt update
```

If `apt update` shows lines starting with `Get:` instead of `Err:` — the network is working.

Update the system:

```bash
apt update && apt upgrade -y
apt install curl -y
```

---

### Step 4 — Install Jellyfin

#### Method A — Official install script (try first)

```bash
curl https://repo.jellyfin.org/install-deb.sh | bash
```

#### Method B — Manual repository method (if the script fails)

```bash
apt install apt-transport-https ca-certificates gnupg curl -y

# Add Jellyfin's GPG key
curl -fsSL https://repo.jellyfin.org/debian/jellyfin_team.gpg.key \
  | gpg --dearmor -o /usr/share/keyrings/jellyfin.gpg

# Add the repository
echo "deb [signed-by=/usr/share/keyrings/jellyfin.gpg] \
  https://repo.jellyfin.org/debian bookworm main" \
  > /etc/apt/sources.list.d/jellyfin.list

# Install
apt update && apt install jellyfin -y

# Enable and start
systemctl enable jellyfin
systemctl start jellyfin
systemctl status jellyfin
# Should show: active (running)
```

---

### Step 5 — Initial Jellyfin Configuration

From a browser on any device on your local network, open:

```
http://<CONTAINER_IP>:8096
```

Replace `<CONTAINER_IP>` with the IP address of your LXC container.

Follow the setup wizard:

1. **Select language** → English (or your preference)
2. **Create admin account** → Choose a username and password
3. **Set up media libraries** → Skip for now (we'll add them in the next step)
4. **Enable remote access** → Leave defaults
5. Finish setup

---

### Step 6 — Add Media Libraries (Smart Filtering)

This is the key step — Jellyfin's **Content Type** setting determines which file types it shows, hiding everything else.

In the Jellyfin dashboard:

1. Go to **Dashboard** (top-right hamburger menu) → **Libraries**
2. Click **Add Media Library**

For each of the three drives, create a library:

#### For Mobile drive (family photos & videos):

| Setting | Value |
|---------|-------|
| **Content type** | `Photos` |
| **Display name** | `Mobile Media` |
| **Folders** | Click `+` → type `/mnt/Mobile` |

Click **OK**.

#### For Computer drive:

| Setting | Value |
|---------|-------|
| **Content type** | `Photos` (or `Movies` if you have films there) |
| **Display name** | `Computer Media` |
| **Folders** | `/mnt/Computer` |

#### For Others drive:

| Setting | Value |
|---------|-------|
| **Content type** | `Photos` |
| **Display name** | `Other Files` |
| **Folders** | `/mnt/Others` |

> **Why "Photos" content type?** When set to `Photos`, Jellyfin scans the directory for image and video files only. All other file types (PDFs, executables, text files, archives) are completely ignored and never shown. This achieves the "clean view" with only media content visible.

After adding libraries, Jellyfin begins scanning automatically. This can take several minutes depending on how many files are on the drives.

---

### Step 7 — Enable DLNA for the TV

DLNA (Digital Living Network Alliance) is the protocol that lets the Sony TV discover and browse your Jellyfin library using just the remote control.

In the Jellyfin dashboard:

1. Go to **Dashboard** → **DLNA** (in the left sidebar under Advanced)
2. Enable **Enable DLNA server**
3. Enable **Enable DLNA playback** (if shown)
4. Click **Save**

---

## Watching on Sony TV with Remote Only

Once DLNA is enabled and the media scan is complete:

1. On the Sony TV, go to the **Home** menu
2. Open **Media Player** app (may be labeled "Video", "Photo", or "Media")
3. Navigate to **Network** or **Connected Devices**
4. Your Jellyfin server will appear — usually named **"Jellyfin"** or your server's hostname
5. Select it and browse your libraries using the arrow keys and OK button on the remote

**What you'll see:**
- Your three drives appear as separate folders
- Only photos and videos are shown — no other files
- Thumbnails are generated automatically for all videos
- No phone, computer, or app required

> **Note:** The first time Jellyfin runs, thumbnail generation takes time proportional to the number of files. Leave it running for a while before checking the TV.

---

## Troubleshooting

### ❌ `apt update` fails — "Temporary failure resolving"

**Cause:** Container DNS points to Pi-hole or Tailscale, which isn't reachable yet.

**Fix:** Add public DNS to `/etc/resolv.conf`:

```
nameserver 8.8.8.8
nameserver 8.8.4.4
```

---

### ❌ Jellyfin install script fails

**Cause:** The official `install-deb.sh` script sometimes has compatibility issues with newer Debian versions.

**Fix:** Use Method B (manual repository) from Step 4. This is more reliable and gives you the same result.

---

### ❌ `/mnt/Mobile` shows "Permission denied" inside the container

**Cause:** NTFS partitions mounted on the host may have root ownership. A non-privileged container cannot read them.

**Fix:** Ensure the container is **Privileged** (Unprivileged = unchecked). If you created it as unprivileged, you need to recreate it.

---

### ❌ Sony TV doesn't show Jellyfin in Media Player

**Checklist:**
1. Verify DLNA is enabled in Jellyfin Dashboard → DLNA
2. Confirm the TV and Jellyfin container are on the **same local network** (same Wi-Fi / LAN)
3. Check that `jellyfin` service is running: `systemctl status jellyfin`
4. Try restarting the TV's Media Player app or the TV itself
5. Verify no firewall is blocking UDP port 1900 (DLNA discovery)

---

### ❌ No thumbnails generated

**Cause:** Thumbnail generation is resource-intensive and runs in the background.

**Fix:** Wait longer (can take 30–60 min for large libraries). Check progress in Jellyfin Dashboard → Activity Log.

If thumbnails are never generated, check that `ffmpeg` is installed:

```bash
apt install ffmpeg -y
systemctl restart jellyfin
```

---

### ❌ Non-media files are still showing up

**Cause:** Wrong Content Type selected for the library.

**Fix:** In Jellyfin Dashboard → Libraries, click the pencil icon on the library and change **Content type** to `Photos`. Then trigger a re-scan.

---

## Architecture Diagram

```
Sony Smart TV (remote control only)
       │
       │  DLNA (UDP 1900 + TCP 8096)
       │  Local Wi-Fi
       ▼
┌─────────────────────────────────────┐
│         Proxmox VE 9.2.3            │
│  ┌─────────────────────────────┐    │
│  │   LXC 102 (Debian 12)       │    │
│  │   Privileged container      │    │
│  │                             │    │
│  │   ┌──────────────────────┐  │    │
│  │   │   Jellyfin           │  │    │
│  │   │   Port: 8096 (web)   │  │    │
│  │   │   DLNA: enabled      │  │    │
│  │   └──────────────────────┘  │    │
│  │                             │    │
│  │   Bind Mounts:              │    │
│  │   /mnt/Mobile  ──────────── │─── │── /dev/sdb1 (NTFS)
│  │   /mnt/Computer ─────────── │─── │── /dev/sdb2 (NTFS)
│  │   /mnt/Others  ──────────── │─── │── /dev/sdb3 (NTFS)
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘

Data flow:
NTFS drives → bind mount → LXC → Jellyfin scans & indexes
→ DLNA broadcast → Sony TV discovers → remote navigation
```

---

## Notes

- The **Jellyfin Nextcloud integration** was evaluated but not implemented in this setup. Nextcloud and Jellyfin use different LXC containers with independent bind mounts to the same physical drives.
- If you want to stream Jellyfin **outside the home**, install Tailscale inside the Jellyfin LXC container (same process as in the Nextcloud documentation) and access `http://<jellyfin-tailscale-ip>:8096`.
- DLNA only works on the **local network**. For remote streaming, use the Jellyfin app (iOS/Android) with Tailscale.
