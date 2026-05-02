# Homelab Setup Guide — Phase 2
## Network Services

**Target:** Create LXC containers, deploy AdGuard Home DNS server, and Nginx Proxy Manager
**Timeframe:** Day 2 (2–3 hours)
**Prerequisite:** [Phase 1 — Proxmox Installation](phase-1-proxmox-installation.md) complete, Proxmox at `https://192.168.1.10:8006`
**Outcome:** CT100 (`network-svc`, 192.168.1.100) running AdGuard Home + Nginx PM; CT101 (`core-svc`, 192.168.1.101) ready for Docker services
**Next:** [Phase 3 — Core Services](phase-3-core-services.md)

---

## Table of Contents

1. [Phase 2 Overview](#phase-2-overview)
2. [SSH Key Setup](#step-21-generate-ssh-key-pair-local-machine)
3. [Add SSH Key to Proxmox](#step-22-add-ssh-key-to-proxmox)
4. [Download LXC Template](#step-23-download-lxc-template)
5. [Create Network Services Container (CT100)](#step-24-create-network-services-container-id-100)
6. [Create Core Services Container (CT101)](#step-25-create-core-services-container-id-101)
7. [Install AdGuard Home](#step-26-install-adguard-home-system-service)
8. [DNS Architecture](#step-27-dns-architecture--pfsense-primary-adguard-backup)
9. [Install Nginx Proxy Manager](#step-28-install-nginx-proxy-manager)
10. [Testing & Verification](#testing--verification-checklist)
11. [Troubleshooting](#troubleshooting)

---

## Phase 2 Overview

### What We're Building

```
┌──────────────────────────────────────────────────────────────────┐
│                        PROXMOX VE                                │
│                    (192.168.1.10:8006)                           │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ LXC CT100: Network Services (1GB RAM) [system service]  │    │
│  │ IP: 192.168.1.100                                       │    │
│  │ ┌──────────────────────────┐  ┌────────────────────────┐│    │
│  │ │ AdGuard Home             │  │ Nginx Proxy Manager    ││    │
│  │ │ DNS port: 53             │  │ HTTP(S) reverse proxy  ││    │
│  │ │ Admin: port 3000         │  │ Port 80, 443, 81       ││    │
│  │ └──────────────────────────┘  └────────────────────────┘│    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ LXC CT101: Core Services (4GB RAM) [Docker-based]       │    │
│  │ IP: 192.168.1.101                                       │    │
│  │ → Phase 3 services go here                              │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### LXC Container Summary

| Container | ID | IP | RAM | Storage | Purpose |
|---|---|---|---|---|---|
| Network Services | CT100 | 192.168.1.100 | 1 GB | 20 GB | AdGuard Home + Nginx PM |
| Core Services | CT101 | 192.168.1.101 | 4 GB | 50 GB | Docker services (Phase 3) |

### DNS Architecture

```
Your devices (DHCP clients)
        │
        ▼ DNS queries
┌───────────────────────────┐
│  pfSense (192.168.1.1)    │  ← PRIMARY DNS
│  pfBlockerNG + Unbound    │    DHCP serves 192.168.1.1 as DNS
│  DNS Resolver             │    Handles: ad-blocking, DNSBL,
│  Host Overrides (.yourdomain.com) │  local split-DNS resolution
└───────────────────────────┘
        │
        ▼ upstream
    Cloudflare 1.1.1.1

┌───────────────────────────┐
│  AdGuard Home (CT100)     │  ← BACKUP only (per-device optional)
│  192.168.1.100 port 53    │    Do NOT set as DHCP DNS server
└───────────────────────────┘
```

---

## SSH Key Setup (Security)

> **Goal:** Generate SSH key pair for secure passwordless access to Proxmox and containers.
> **Why:** Passwords are weak. SSH keys are required for CI/CD automation in later phases.

### Step 2.1: Generate SSH Key Pair (Local Machine)

#### 2.1.1: Check if You Already Have Keys

Open terminal/PowerShell on your laptop:

**Linux/Mac:**
```bash
ls -la ~/.ssh/
```

**Windows (PowerShell):**
```powershell
ls $env:USERPROFILE\.ssh\
```

**Look for files:**
- `id_ed25519` (private key) — keep this SECRET
- `id_ed25519.pub` (public key) — can be shared

**If both exist:** Skip to Step 2.1.4.

#### 2.1.2: Generate New SSH Key Pair

> **Algorithm:** Use `ed25519` — current industry best practice. Smaller key, faster, more secure than RSA-4096. Use RSA only if the remote host is too old to support ed25519 (rare).

**Linux/Mac:**
```bash
ssh-keygen -t ed25519 -C "your-device-name"
```

**Windows (PowerShell):**
```powershell
ssh-keygen -t ed25519 -C "your-device-name"
```

**Prompts:**
```
Enter passphrase (empty for no passphrase): [LEAVE BLANK or set one for extra security]
Enter same passphrase again: [confirm]
```

#### 2.1.3: Verify Keys Generated

**Linux/Mac:**
```bash
cat ~/.ssh/id_ed25519.pub
```

**Windows (PowerShell):**
```powershell
cat $env:USERPROFILE\.ssh\id_ed25519.pub
```

**Output shows public key:**
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5... your-device-name
```

**Copy this entire line — you'll paste it into Proxmox.**

**⚠️ IMPORTANT:**
- **Private key** (`id_ed25519`) → Keep SECRET, never share, never commit to Git
- **Public key** (`id_ed25519.pub`) → Safe to share, paste into servers

---

### Step 2.2: Add SSH Key to Proxmox

#### 2.2.1: Create SSH Directory in Proxmox

In Proxmox shell (web UI → Nodes → homelab → Shell):

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

#### 2.2.2: Add Your Public Key

```bash
cat >> ~/.ssh/authorized_keys << 'EOF'
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5... your-device-name
EOF
```

**Replace `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5...` with your actual public key from Step 2.1.3**

> **Multiple devices:** `authorized_keys` holds one key per line and supports any number of keys. The `>>` operator appends without overwriting existing entries — repeat this command for each device.

#### 2.2.3: Set Correct Permissions

```bash
chmod 600 ~/.ssh/authorized_keys
```

#### 2.2.4: Test SSH Connection from Laptop

**Linux/Mac:**
```bash
ssh root@192.168.1.10
```

**Windows (PowerShell):**
```powershell
ssh root@192.168.1.10
```

**First time:**
```
The authenticity of host '192.168.1.10' can't be established.
Are you sure you want to continue connecting? (yes/no/[fingerprint]):
```

**Type:** `yes`

**Expected:** Logged in without entering a password ✅

#### 2.2.5: Add Keys for Additional Devices

For each extra device (laptop, desktop, phone via Termux, etc.), get its public key and append it to the same file on every host you want it to access:

```bash
# On the new device — get the public key
cat ~/.ssh/id_ed25519.pub
# (or id_rsa.pub if RSA)
```

Then on each server (Proxmox host and/or containers), append it:

```bash
cat >> ~/.ssh/authorized_keys << 'EOF'
ssh-ed25519 AAAAC3NzaC1lZDI1... device-name
EOF
```

Verify all keys are present:

```bash
cat ~/.ssh/authorized_keys
# Each line is one key — one per device
```

To revoke a device's access, delete its line from `authorized_keys`.

---

## Create LXC Containers

### Step 2.3: Download LXC Template

Before creating containers, download the Debian OS template in Proxmox:

1. Proxmox web UI → left sidebar → **homelab** (your node) → **local** storage → **CT Templates**
2. Click **Templates** button (top of content area)
3. In the search box type: `debian-12`
4. Select **Debian 12 Bookworm Standard** → Click **Download**
5. Wait for download to complete (progress in Task Log at bottom)

> **Why:** Without this step, the OS dropdown in the CT creation wizard is empty.

---

### Step 2.4: Create Network Services Container (ID: 100)

#### 2.4.1: In Proxmox Web UI

1. Left sidebar → **Nodes** → **homelab** → **Create CT** (top right button)

```
┌──────────────────────────────────────────┐
│ Create: LXC Container                    │
│                                          │
│ Hostname:     [network-svc]              │
│ CT ID:        [100]                      │
│ Password:     [••••••••••••]             │
│ Root SSH Key: [copy/paste your key]      │
│                                          │
│ [Next] [Cancel]                          │
└──────────────────────────────────────────┘
```

**Fill in:**
- **Hostname:** `network-svc`
- **CT ID:** `100`
- **Password:** Set a strong root password (e.g., `NetworkSvc@2026!Labs`)
- **Root SSH Key:** Paste your public key from Step 2.1.3

Click **Next**

#### 2.4.2: Select OS

Keep **Debian 12 (bookworm)** → Click **Next**

#### 2.4.3: Disk Configuration

- **Storage:** `local-lvm`
- **Size:** `20` GB

Click **Next**

#### 2.4.4: CPU Configuration

- **Cores:** `2` (keep default)

Click **Next**

#### 2.4.5: Memory Configuration

- **Memory (MB):** `1024` (1 GB)
- **Swap (MB):** `512`

Click **Next**

#### 2.4.6: Network Configuration

- **IP Address:** `192.168.1.100/24`
- **Gateway:** `192.168.1.1`

Click **Next**

#### 2.4.7: DNS Configuration

Keep defaults (DNS: 8.8.8.8, 8.8.4.4). Click **Next**

#### 2.4.8: Summary & Confirm

```
┌──────────────────────────────────────────┐
│ Summary                                  │
│                                          │
│ CT ID:        100                        │
│ Hostname:     network-svc                │
│ OS:           Debian 12                  │
│ CPU:          2 cores                    │
│ RAM:          1024 MB                    │
│ Disk:         20 GB                      │
│ IP:           192.168.1.100/24           │
│ Gateway:      192.168.1.1                │
│                                          │
│ [Finish] [Cancel]                        │
└──────────────────────────────────────────┘
```

Review carefully, then click **Finish**. Container creation takes ~2–3 minutes.

#### 2.4.9: Start the Container

Once creation finishes → **network-svc (100)** → **Start**

Verify:
```bash
pct status 100
# Expected: running
```

---

### Step 2.5: Create Core Services Container (ID: 101)

#### 2.5.1: Repeat Container Creation

1. Proxmox web UI → **Nodes** → **homelab** → **Create CT**
2. Follow the same steps as 2.4.1–2.4.8, but change:

| Field | Value |
|---|---|
| Hostname | `core-svc` |
| CT ID | `101` |
| Size | `50` GB (Docker storage needs) |
| Memory | `4096` MB (4 GB — services are RAM-heavy) |
| IP Address | `192.168.1.101/24` |

Everything else stays the same.

#### 2.5.2: Start Container 101

```bash
pct status 101
# Expected: running
```

---

## Network Services LXC

### Step 2.6: Install AdGuard Home (System Service)

> **Goal:** DNS server with ad-blocking, running as system service (not Docker).
> **Why system service:** DNS is critical; it must not depend on Docker daemon health.

> **Architecture note:** AdGuard is placed in **CT100 (network-svc)**, not CT101. DNS should be separate from Docker-based services.

#### 2.6.1: Access Container 100 Shell

```bash
ssh root@192.168.1.100
# OR via Proxmox: pct enter 100
```

#### 2.6.2: Update System

```bash
apt update && apt upgrade -y
```

#### 2.6.3: Download & Install AdGuard Home

```bash
cd /tmp
curl -L https://github.com/AdguardTeam/AdGuardHome/releases/download/v0.107.46/AdGuardHome_linux_amd64.tar.gz -o adguard.tar.gz
tar -xzf adguard.tar.gz
```

> **Check for latest version:** https://github.com/AdguardTeam/AdGuardHome/releases — replace `v0.107.46` if newer.

#### 2.6.4: Install to System Directory

```bash
mkdir -p /opt/AdGuardHome
mv /tmp/AdGuardHome/AdGuardHome /opt/AdGuardHome/
chmod +x /opt/AdGuardHome/AdGuardHome
```

#### 2.6.5: Create AdGuard Home Data Directory

```bash
mkdir -p /var/lib/adguardhome
chmod 755 /var/lib/adguardhome
```

#### 2.6.6: Create Systemd Service File

```bash
cat > /etc/systemd/system/adguardhome.service << 'EOF'
[Unit]
Description=AdGuard Home
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/var/lib/adguardhome
ExecStart=/opt/AdGuardHome/AdGuardHome -c /var/lib/adguardhome/AdGuardHome.yaml -w /var/lib/adguardhome
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```

#### 2.6.7: Start AdGuard Home Service

```bash
systemctl daemon-reload
systemctl enable adguardhome
systemctl start adguardhome
```

#### 2.6.8: Verify Service is Running

```bash
systemctl status adguardhome
```

**Expected:**
```
● adguardhome.service - AdGuard Home
   Active: active (running) since ...
```

#### 2.6.9: Access AdGuard Home Web UI

From laptop browser:
```
http://192.168.1.100:3000
```

**First-time setup wizard:**
1. Change Admin Interface to: `0.0.0.0:3000` (accessible from laptop)
2. Keep DNS Server Bind: `0.0.0.0:53` (this controls what port AdGuard *listens* on — leave as-is)
3. Click **Configure**

**Set admin credentials:**
- Username: `admin`
- Password: `AdGuard@2026!Home` (or your own)

Click **Next** → **Finish**

#### 2.6.10: Fix Upstream DNS Servers

After the wizard, the **Upstream DNS servers** list may contain `0.0.0.0:53` — this is wrong (it forwards queries back to itself, causing a loop). Fix it:

1. AdGuard Home → **Settings** → **DNS settings**
2. Under **Upstream DNS servers**, clear the field and replace with:
   ```
   https://dns10.quad9.net/dns-query
   https://dns.cloudflare.com/dns-query
   ```
3. Remove `0.0.0.0:53` if present
4. Click **Apply**

> These are DNS-over-HTTPS (DoH) upstreams — encrypted so your ISP can't see your DNS queries. Quad9 also blocks malware domains. You can use any upstream from the [AdGuard known providers list](https://kb.adguard.com/en/general/dns-providers).

#### 2.6.11: Configure Local DNS Overrides in pfSense

> **Why pfSense, not AdGuard, for host overrides:**
> 1. **Traffic flow** — pfSense is your DHCP server, so it hands out `192.168.1.1` as the DNS server to every device on your LAN. All DNS queries go to pfSense first. AdGuard is only consulted by devices manually configured to use it — so overrides in AdGuard are invisible to the rest of your network.
> 2. **Failure isolation** — pfSense is your router; it's always on. AdGuard lives in an LXC container that can be restarted, updated, or crash. If AdGuard goes down and your overrides live there, `vault.yourdomain.com` stops resolving internally and falls back to Cloudflare's public IP — which either doesn't exist yet or routes traffic externally instead of staying on your LAN.
> 3. **Single source of truth** — keeping all internal DNS in one place (pfSense) means adding a new service is one step, not two. Overrides split across pfSense and AdGuard can silently diverge.

**Verify you're in the right place before adding anything:**

| | pfSense ✅ (correct) | AdGuard ❌ (wrong) |
|---|---|---|
| **URL** | `http://192.168.1.1` | `http://192.168.15.100:3000` |
| **Page title** | pfSense — Services / DNS Resolver | AdGuard Home — Filters / DNS rewrites |
| **Navigation path** | Services → DNS Resolver → Host Overrides | Filters → DNS rewrites |
| **Form fields** | Host, Domain, IP address, Description | Domain, Answer (IP or domain) |

If you see **"DNS rewrites"** in the nav — you're in AdGuard, close that tab and go to `http://192.168.1.1` instead.

1. Log into pfSense at `http://192.168.1.1`
2. Go to **Services** → **DNS Resolver**
3. Scroll to the bottom to **Host Overrides** section → click **+ Add**

Add each entry:

| Host | Domain | IP | Description |
|---|---|---|---|
| `homelab` | `yourdomain.com` | `192.168.1.10` | Proxmox management |
| `adguard` | `yourdomain.com` | `192.168.1.100` | AdGuard Home admin |
| `nginx` | `yourdomain.com` | `192.168.1.100` | Nginx Proxy Manager |
| `vault` | `yourdomain.com` | `192.168.1.100` | Vaultwarden (via Nginx) |
| `immich` | `yourdomain.com` | `192.168.1.100` | Immich (via Nginx) |
| `pdf` | `yourdomain.com` | `192.168.1.100` | Stirling-PDF (via Nginx) |
| `status` | `yourdomain.com` | `192.168.1.100` | Uptime Kuma (via Nginx) |
| `portainer` | `yourdomain.com` | `192.168.1.101` | Portainer (via Nginx) |

For each entry: **Host** + **Domain** + **IP** → **Save** → when all done, click **Apply Changes**

**Verify resolution from laptop:**
```bash
nslookup vault.yourdomain.com 192.168.1.1
# Should return 192.168.1.100 (your internal Nginx PM IP, NOT Cloudflare's public IP)
```

> **This is split-DNS in action:** pfSense intercepts `vault.yourdomain.com` and returns the internal IP. The query never reaches Cloudflare's public DNS.

---

### Step 2.7: DNS Architecture — pfSense Primary, AdGuard Backup

> **IMPORTANT:** Read carefully before changing any pfSense DNS settings.

#### 2.7.1: DNS Role Summary

| Component | Role | IP | Port |
|---|---|---|---|
| pfSense Unbound | **Primary DNS** (DHCP hands this out) | 192.168.1.1 | 53 |
| pfBlockerNG | Ad-blocking DNSBL (runs inside Unbound) | – | – |
| AdGuard Home | **Secondary/Backup** (per-device optional) | 192.168.1.100 | 53 |

**Rule:** pfSense DHCP must serve `192.168.1.1` as DNS. AdGuard is for specific devices only.

#### 2.7.2: Verify pfSense DHCP DNS Is Correct

1. pfSense web UI → **Services** → **DHCP Server** → **LAN**
2. Scroll to **Servers** section
3. Verify **DNS Servers** is **empty** OR `192.168.1.1`
   - Empty = clients get pfSense IP automatically ✅
   - If it shows `192.168.1.100` → **change it back to empty or `192.168.1.1`**

> ⚠️ **Do NOT set DNS to `192.168.1.100` (AdGuard) in DHCP.** That bypasses pfBlockerNG and breaks ad-blocking + internal `*.yourdomain.com` split-DNS resolution.

#### 2.7.3: Optional — Point Specific Devices to AdGuard

If you want one device to use AdGuard Home instead of pfSense:

1. pfSense → **Services** → **DHCP Server** → **DHCP Static Mappings**
2. Add static mapping for that device's MAC address
3. Set **DNS Servers** to `192.168.1.100` for that specific device only

All other devices remain on pfSense DNS.

---

### Step 2.8: Install Nginx Proxy Manager

> **Goal:** Reverse proxy that routes traffic from clean URLs to internal services + handles SSL.
> **Why:** Services are on different ports (8080, 2283, 8081, etc.). Nginx unifies them behind clean `*.yourdomain.com` URLs — the same URLs that will work externally via Cloudflare Tunnel in Phase 4.

#### 2.8.1: Install Docker in Network Services Container

SSH into container 100:

```bash
ssh root@192.168.1.100
```

Install Docker:

```bash
apt update && apt install -y docker.io docker-compose
systemctl enable docker
systemctl start docker
```

#### 2.8.2: Verify Docker Works

```bash
docker --version
docker run hello-world
```

**Expected:** Container runs and prints "Hello from Docker!"

#### 2.8.3: Create Docker Compose for Nginx PM

```bash
mkdir -p /opt/nginx-pm
cd /opt/nginx-pm
```

Create `docker-compose.yml`:

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: always
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
    environment:
      DB_SQLITE_FILE: "/data/database.sqlite"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    networks:
      - npm-network

networks:
  npm-network:
    driver: bridge
EOF
```

#### 2.8.4: Start Nginx PM

```bash
docker-compose up -d
```

Wait ~30 seconds for the container to start.

#### 2.8.5: Access Nginx PM Admin Panel

From laptop browser:
```
http://192.168.1.100:81
```

**Default login:**
```
Email:    admin@example.com
Password: changeme
```

**Change credentials immediately:**
1. Top right → **User** → **Change password**
2. Set strong password: `NginxPM@2026!Admin`

✅ Nginx PM is running and ready. Proxy hosts will be added in Phase 3 once services are deployed.

---

## Testing & Verification Checklist

```
[ ] SSH key works: ssh root@192.168.1.10 (no password prompt)
[ ] CT100 running: pct status 100 → shows "running"
[ ] CT101 running: pct status 101 → shows "running"
[ ] AdGuard Home accessible: http://192.168.1.100:3000
[ ] AdGuard systemd service active: systemctl status adguardhome
[ ] pfSense host overrides added and applied
[ ] nslookup vault.yourdomain.com 192.168.1.1 → returns 192.168.1.100 (internal IP, not Cloudflare)
[ ] pfSense DHCP DNS is 192.168.1.1 (NOT 192.168.1.100)
[ ] Nginx PM accessible: http://192.168.1.100:81
[ ] Nginx PM default credentials changed
```

---

## Troubleshooting

### Problem: `pct enter 100` fails or container won't start

**Solutions:**
1. Check Proxmox storage has space: `df -h /` on Proxmox host
2. Check container logs: `journalctl -u pve-container@100`
3. Destroy and recreate: `pct stop 100 && pct destroy 100 --purge` → repeat Step 2.4

---

### Problem: AdGuard Home not accessible at :3000

**Solutions:**
1. Check service: `systemctl status adguardhome`
2. Check if port is listening: `ss -tlnp | grep 3000`
3. Check AdGuard logs: `journalctl -u adguardhome -n 50`
4. Check container firewall: `iptables -L` (should have no DROP rules blocking 3000)

---

### Problem: `*.yourdomain.com` internal domains not resolving after adding pfSense host overrides

**Solutions:**
1. Verify "Apply Changes" was clicked in pfSense DNS Resolver
2. Flush DNS cache on laptop: `ipconfig /flushdns` (Windows) or `sudo dscacheutil -flushcache` (Mac)
3. Test from pfSense itself: pfSense → **Diagnostics** → **DNS Lookup** → type `vault.yourdomain.com`
4. Verify pfSense Unbound is running: pfSense → **Services** → **DNS Resolver** → check status

---

### Problem: Nginx PM default credentials rejected

**Cause:** Container may not be fully initialized yet

**Solutions:**
1. Wait 30–60 seconds, try again
2. Check container: `docker logs nginx-proxy-manager`
3. Restart container: `docker-compose restart` in `/opt/nginx-pm`

---

## Next Steps

**Phase 2 complete!** Both containers are running, DNS is configured, and Nginx PM is ready.

**Phase 3 — Core Services:**
- Install Docker in CT101
- Deploy Portainer, Vaultwarden, Immich, Stirling-PDF, Uptime Kuma
- Configure B2 backup for Vaultwarden
- Add proxy hosts in Nginx PM for all services

See: [phase-3-core-services.md](phase-3-core-services.md)

---

*Last updated: 2026-05-01*
*Based on: HP EliteDesk 800 G4 SFF (i5-8500, 16GB DDR4, 500GB NVMe)*
*Network: pfSense 2.8.1-RELEASE/amd64 (FreeBSD 15.0-CURRENT) → CT100 (192.168.1.100) → CT101 (192.168.1.101)*
