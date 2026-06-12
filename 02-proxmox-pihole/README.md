# 🛡️ Proxmox + Pi-hole (Network-Wide Ad Blocker)

> **Pi-hole** deployed in a dedicated **Debian 12 LXC container** on Proxmox VE, providing DNS-level ad blocking for every device on the home network — including Smart TVs, phones, and game consoles.

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Why a Separate LXC](#why-a-separate-lxc)
3. [Prerequisites](#prerequisites)
4. [Step-by-Step Setup](#step-by-step-setup)
   - [Step 1 — Download Debian Template](#step-1--download-debian-template)
   - [Step 2 — Create the LXC Container](#step-2--create-the-lxc-container)
   - [Step 3 — Enable Nesting](#step-3--enable-nesting)
   - [Step 4 — Fix DNS for Installation](#step-4--fix-dns-for-installation)
   - [Step 5 — Install Pi-hole](#step-5--install-pi-hole)
   - [Step 6 — Change Admin Password](#step-6--change-admin-password)
   - [Step 7 — Add Blocklists](#step-7--add-blocklists)
   - [Step 8 — Configure Router DNS](#step-8--configure-router-dns)
5. [Recommended Blocklists](#recommended-blocklists)
6. [Accessing Pi-hole Remotely via Tailscale](#accessing-pi-hole-remotely-via-tailscale)
7. [Troubleshooting](#troubleshooting)

---

## Overview

Pi-hole acts as a **DNS sinkhole** — when any device on your network tries to load an ad, tracker, or malicious domain, Pi-hole intercepts the DNS request and returns nothing, effectively blocking it before any data is downloaded.

**What Pi-hole blocks:**
- Website ads (banners, popups)
- In-app ads on phones and Smart TVs
- Tracking pixels and analytics
- Malware domains

**What Pi-hole does NOT block:**
- YouTube video ads (YouTube serves ads from the same domain as videos — DNS blocking would break playback)

---

## Why a Separate LXC

Pi-hole requires exclusive control of **port 53** (DNS) and optionally **port 80** (its web admin panel). Installing it alongside Nextcloud AIO in the same container would cause port conflicts.

A dedicated LXC container ensures:
- Clean port allocation with no conflicts
- Independent restart/update cycle
- Minimal resource usage (~50–80 MB RAM)

---

## Prerequisites

- Proxmox VE installed and running
- Nextcloud or other services already configured (optional)
- Tailscale installed on the Proxmox host (optional, for remote access)

---

## Step-by-Step Setup

### Step 1 — Download Debian Template

In the Proxmox web UI:

1. Click **local** (storage) in the left sidebar
2. Click **CT Templates** → **Templates**
3. Search for `debian-12-standard` and click **Download**
4. Wait for `TASK OK`

---

### Step 2 — Create the LXC Container

Click **Create CT** and fill in:

| Tab | Setting | Value |
|-----|---------|-------|
| **General** | Hostname | `pihole` |
| | Unprivileged container | ✅ **Checked** (Pi-hole doesn't need NTFS access) |
| | Password | Choose a strong root password |
| **Template** | Template | `debian-12-standard` |
| **Disks** | Disk size | `8 GB` |
| **CPU** | Cores | `1` |
| **Memory** | RAM | `512 MB` |
| | Swap | `512 MB` |
| **Network** | IPv4 | **Static** — e.g. `192.168.0.92/24` |
| | Gateway | Your router IP (e.g. `192.168.0.1`) |

> **Why Static IP?** Pi-hole serves as your DNS server. If its IP changes, all devices lose DNS resolution.

Click **Finish** but **do not start yet**.

---

### Step 3 — Enable Nesting

Debian 12 uses systemd 252, which requires Nesting to run correctly in LXC.

After container creation, enable it via the UI:

1. Click the container in the left sidebar
2. Go to **Options** → **Features** → **Edit**
3. Check **Nesting** → **OK**
4. Start the container

Or manually in the Proxmox host shell:

```bash
# Replace 100 with your container ID
nano /etc/pve/lxc/100.conf
# Add this line:
features: nesting=1
```

Restart the container after saving.

---

### Step 4 — Fix DNS for Installation

Open the container **Console** and log in as root.

When using Tailscale on the Proxmox host, the container may inherit Tailscale's DNS (`100.100.100.100`), which can block access to Debian package servers during installation.

Fix it temporarily:

```bash
nano /etc/resolv.conf
```

Add these lines at the **top** of the file (before existing content):

```
nameserver 8.8.8.8
nameserver 8.8.4.4
```

Save and test:

```bash
apt update
```

You should now see package lists downloading successfully.

Update the system fully:

```bash
apt upgrade -y
apt install curl -y
```

---

### Step 5 — Install Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

The installer shows a blue text-based wizard. Key choices:

| Prompt | Recommended Choice |
|--------|--------------------|
| Static IP detected | **OK** |
| Upstream DNS provider | **Custom** → enter `100.100.100.100` (Tailscale DNS) or `8.8.8.8` |
| Select protocols | **IPv4** (and IPv6 if you use it) |
| Install web interface | **Yes** |
| Install web server (lighttpd) | **Yes** |
| Enable query logging | **Yes** |

> **Using Tailscale DNS (`100.100.100.100`) as upstream** is preferred if Tailscale is your VPN, as it keeps DNS resolution inside your private network and avoids relying on Google.

At the end, the installer shows a **green summary screen** with:
- Your Pi-hole's IP address
- A randomly generated admin password

**Save this password immediately.**

---

### Step 6 — Change Admin Password

If the auto-generated password doesn't work (copy-paste issues are common), reset it:

```bash
pihole -a -p
# Enter and confirm your new password
```

Access the web admin panel:

```
http://192.168.0.92/admin
```

Log in with your password.

---

### Step 7 — Add Blocklists

In the Pi-hole admin panel:

1. Go to **Adlists** in the left sidebar
2. Paste a blocklist URL in the **Address** field
3. Add a **Comment** (e.g., `StevenBlack_Main`)
4. Click **Add**
5. After adding all lists, go to **Tools** → **Update Gravity** (or run `pihole -g` in the console)

See [Recommended Blocklists](#recommended-blocklists) below.

---

### Step 8 — Configure Router DNS

For Pi-hole to filter traffic for **all devices** on the network, your router must point to it as the DNS server.

#### Option A — Set DNS in Router (if supported)

Log in to your router's admin page and set the **Primary DNS** to Pi-hole's IP:

```
192.168.0.92
```

> ⚠️ **Vodafone Station limitation:** The Vodafone Station router does not expose DNS settings in its UI. Use Option B instead.

#### Option B — Enable Pi-hole as DHCP Server (Vodafone workaround)

1. In the Vodafone Station admin, find **LAN / DHCP** settings and **disable the DHCP server**
2. In the Pi-hole admin panel → **Settings** → **DHCP**:
   - Enable **DHCP server**
   - Set IP range: e.g., `192.168.0.100` to `192.168.0.200`
   - Set **Router (gateway)** to your Vodafone Station IP
   - Save

Pi-hole now assigns IPs to all devices and tells them to use `192.168.0.92` as DNS.

#### Option C — Per-device Manual DNS

Set `192.168.0.92` as the DNS server manually on each device:

- **Windows:** Network Adapter → Properties → IPv4 → Preferred DNS
- **Android/iPhone:** Wi-Fi settings → Modify network → Static IP → DNS 1

---

## Recommended Blocklists

Start with these two — they cover the vast majority of ads and trackers without breaking legitimate sites:

| List | URL | Notes |
|------|-----|-------|
| **StevenBlack Unified** | `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` | The gold standard. General ads + tracking |
| **OISD Big** | `https://big.oisd.nl` | Curated, low false-positive rate. Ads + malware |

Optional additions (add carefully — may cause false positives):

| List | URL | Purpose |
|------|-----|---------|
| Hagezi Pro | `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/pro.txt` | Advanced tracking |
| StevenBlack Porn | `https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/porn/hosts` | Adult content blocking |
| URLHaus | `https://urlhaus.abuse.ch/downloads/hostfile/` | Malware domains |

> **Tip:** After adding lists, always run **Update Gravity** (`pihole -g`) for changes to take effect.

---

## Accessing Pi-hole Remotely via Tailscale

Pi-hole's web panel is available inside the Tailscale network at:

```
http://100.x.x.x/admin
```

Where `100.x.x.x` is the Tailscale IP of the Pi-hole container.

For DNS filtering to apply when **outside the home**, configure Tailscale to use Pi-hole as its DNS server:

1. Open **Tailscale admin panel** → **DNS**
2. Add a custom nameserver: `192.168.0.92` (Pi-hole's local IP)
3. Enable **Override local DNS**

This routes all Tailscale-connected devices through Pi-hole for DNS — blocking ads even when you're on mobile data.

---

## Troubleshooting

### ❌ `apt update` fails — "Temporary failure resolving"

**Cause:** The container inherited Tailscale's DNS (`100.100.100.100`), which may not resolve public domains before Tailscale is active.

**Fix:** Add public DNS at the top of `/etc/resolv.conf`:

```
nameserver 8.8.8.8
nameserver 8.8.4.4
```

---

### ❌ `curl` is not available

**Fix:**

```bash
apt install curl -y
```

Then re-run the Pi-hole installer.

---

### ❌ Pi-hole installer warns about Systemd 252 / Nesting

**Fix:** Enable Nesting in Proxmox Options → Features, then restart the container before installing.

---

### ❌ Admin panel says "Wrong password"

**Fix:** Reset the password from the container console:

```bash
pihole -a -p
```

Then try again from the browser in an **Incognito/Private window** to avoid cached credentials.

---

### ❌ Blocking too aggressive — legitimate sites broken

**Fix:** In the Pi-hole admin panel → **Query Log**, find the blocked domain, and click **Whitelist** to allow it.

Or add it via console:

```bash
pihole -w example.com
```
