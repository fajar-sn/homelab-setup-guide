# Homelab Setup Guide

A step-by-step personal homelab build guide for an **HP EliteDesk 800 G4 SFF** running behind **IndiHome CGNAT**, using **Proxmox VE** as the hypervisor and **Tailscale** for zero-config remote access.

---

## Project Architecture

```
Internet (IndiHome — CGNAT, no public IP)
        │
        ▼
    Your Router (pfSense — 192.168.1.1)
        │  DHCP, DNS, pfBlockerNG, split-DNS for *.yourdomain.com
        │
        ▼
    Proxmox VE Host (192.168.1.10:8006)
    HP EliteDesk 800 G4 SFF — NVMe-only
    Tailscale subnet router (100.x.x.x mesh)
        │
        ├── LXC CT100: Network Services (192.168.1.100)
        │       ├── AdGuard Home     — port 53 / admin :3000
        │       └── Nginx Proxy Manager — ports 80, 443, 81
        │
        └── LXC CT101: Core Services (192.168.1.101)
                └── Docker
                        ├── Portainer CE       — :9000 / :9443
                        ├── Vaultwarden        — :8080  → vault.yourdomain.com
                        ├── Immich             — :2283  → immich.yourdomain.com
                        ├── Stirling-PDF       — :8081  → pdf.yourdomain.com
                        └── Uptime Kuma        — :3001  → status.yourdomain.com
```

### Remote Access

All `*.yourdomain.com` URLs are **internal-only**. Remote access is provided by Tailscale — no open ports, no public IP required (CGNAT-safe).

```
Remote device (phone / laptop)
    │  Tailscale ON
    ▼
Tailscale mesh (100.x.x.x) ──► Proxmox subnet router
    │  route: 192.168.1.0/24
    ▼
pfSense DNS: vault.yourdomain.com → 192.168.1.100
    ▼
Nginx Proxy Manager → CT101 Docker service
```

### Hardware & Storage

| Component | Spec |
|---|---|
| Machine | HP EliteDesk 800 G4 SFF |
| RAM | 16 GB DDR4 |
| Storage | NVMe SSD (no HDD — starter config) |
| Network | 1 GbE onboard |

| Container | ID | IP | RAM | Disk | Purpose |
|---|---|---|---|---|---|
| Network Services | CT100 | 192.168.1.100 | 1 GB | 20 GB | AdGuard Home + Nginx PM |
| Core Services | CT101 | 192.168.1.101 | 4 GB | 50 GB | Docker workloads |

---

## Setup Phases

| Phase | Guide | Status |
|---|---|---|
| Phase 0 — Preparation | [docs/phase-0-preparation.md](docs/phase-0-preparation.md) | |
| Phase 1 — Proxmox Installation | [docs/phase-1-proxmox-installation.md](docs/phase-1-proxmox-installation.md) | |
| Phase 2 — Network Services | [docs/phase-2-network-services.md](docs/phase-2-network-services.md) | |
| Phase 3 — Core Services | [docs/phase-3-core-services.md](docs/phase-3-core-services.md) | |
| Phase 4 — Remote Access (Tailscale) | [docs/phase-4-remote-access.md](docs/phase-4-remote-access.md) | |

For full architecture notes, trade-off analysis, and the software decision guide, see [docs/homelab-setup-guide.md](docs/homelab-setup-guide.md).

---

## Continuing Setup on a New / Different Client Device

If you're picking this up from a different laptop, desktop, or fresh OS install:

### 1. Clone this repo

```bash
git clone https://github.com/fajar-sn/homelab-setup-guide.git
cd homelab-setup-guide
```

### 2. Install Tailscale

Download and install from **https://tailscale.com/download** for your OS, then sign in with the **same account** used when setting up the server.

```bash
# Linux
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Once connected, your device gets a `100.x.x.x` address and the subnet route `192.168.1.0/24` becomes reachable.

### 3. Set up SSH key on this device

#### Algorithm note

Use **ed25519** — it is the current industry best practice. It produces smaller keys, is faster, and is more resistant to side-channel attacks than RSA. RSA-4096 is still acceptable on legacy systems that do not support ed25519, but there is no reason to prefer it on modern hardware.

| Algorithm | Recommendation | Notes |
|---|---|---|
| `ed25519` | ✅ Use this | Current standard, fast, compact |
| `rsa -b 4096` | ⚠️ Legacy fallback | Only if the remote host is very old |
| `ecdsa` | ✅ Acceptable | Rarely needed; ed25519 is preferred |
| `dsa` / `rsa -b 1024` | ❌ Do not use | Cryptographically broken |

#### Option A — Generate a new key on this device

```bash
ssh-keygen -t ed25519 -C "your-device-name"
```

Get your new public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the entire output line. Then append it to `authorized_keys` on each host (this adds the key without removing existing ones):

```bash
# Proxmox host
ssh-copy-id root@192.168.1.10

# CT100 — Network Services
ssh-copy-id root@192.168.1.100

# CT101 — Core Services
ssh-copy-id root@192.168.1.101
```

> `ssh-copy-id` appends your public key to `~/.ssh/authorized_keys` on the remote host — it does not overwrite existing keys. All previously authorized devices remain working.
>
> If password authentication has already been disabled on the server (hardened setup), you will need to append the key manually from a device that already has access — see [Step 2.2.5 in Phase 2](docs/phase-2-network-services.md).

#### Option B — Reuse your existing key from another device

Copy `~/.ssh/id_ed25519` (private) and `~/.ssh/id_ed25519.pub` (public) from your old device via a password manager or encrypted transfer (e.g., Vaultwarden → secure note). No server changes needed — the public key is already in `authorized_keys`.

### 4. Verify LAN / Tailscale connectivity

```bash
# Proxmox web UI
open https://192.168.1.10:8006   # or navigate manually

# Ping containers
ping 192.168.1.100
ping 192.168.1.101
```

### 5. Access service dashboards

| Service | URL |
|---|---|
| Proxmox VE | https://192.168.1.10:8006 |
| Nginx Proxy Manager | http://192.168.1.100:81 |
| AdGuard Home | http://192.168.1.100:3000 |
| Portainer | http://portainer.yourdomain.com or http://192.168.1.101:9000 |
| Vaultwarden | http://vault.yourdomain.com |
| Immich | http://immich.yourdomain.com |
| Uptime Kuma | http://status.yourdomain.com |

> Replace `yourdomain.com` with your actual domain. All URLs require either being on your home LAN or Tailscale connected.

---

## After Restarting the Server

All LXC containers and Docker services are configured to start automatically. After a hard reboot or power cut, verify services have come back up cleanly.

### 1. Confirm Proxmox has booted

```bash
ssh root@192.168.1.10
```

Check that both containers are running:

```bash
pct list
```

Expected output:

```
VMID  Status   Name
100   running  network-svc
101   running  core-svc
```

If a container shows `stopped`, start it:

```bash
pct start 100
pct start 101
```

### 2. Verify Docker is running in CT101

```bash
ssh root@192.168.1.101
docker ps
```

All containers (`portainer`, `vaultwarden`, `immich_*`, `stirling-pdf`, `uptime-kuma`) should show `Up`. If Docker itself failed to start:

```bash
systemctl start docker
docker ps
```

### 3. Verify Tailscale is active on the Proxmox host

```bash
ssh root@192.168.1.10
tailscale status
```

The host should show as connected. If it shows disconnected:

```bash
systemctl start tailscaled
tailscale up
```

Then re-check that the subnet route `192.168.1.0/24` is still advertised:

```bash
tailscale status --json | grep -i subnet
```

If the route dropped, re-advertise it:

```bash
tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false
```

Then go to **https://login.tailscale.com/admin/machines**, find the homelab node, and re-approve the subnet route if prompted.

### 4. Verify DNS is resolving

From any LAN device (or via Tailscale):

```bash
nslookup vault.yourdomain.com 192.168.1.1
```

Expected: resolves to `192.168.1.100` (Nginx Proxy Manager).

If DNS is broken, check pfSense → **Services → DNS Resolver → Host Overrides** and verify the `*.yourdomain.com` entries are intact.

### 5. Quick service health check

Open **http://status.yourdomain.com** (Uptime Kuma) — all monitored services should show green within ~2 minutes of the server booting. This is the fastest single-screen confirmation that everything is healthy.

### Post-restart checklist

- [ ] `pct list` — both CT100 and CT101 show `running`
- [ ] `docker ps` on CT101 — all containers `Up`
- [ ] `tailscale status` on Proxmox host — shows connected
- [ ] Subnet route `192.168.1.0/24` approved in Tailscale admin
- [ ] Uptime Kuma dashboard — all services green
- [ ] `vault.yourdomain.com` loads in browser (Vaultwarden — most critical)

---

## Key IPs & Ports Reference

| Host | Address | Access |
|---|---|---|
| pfSense router | 192.168.1.1 | LAN only |
| Proxmox VE | 192.168.1.10:8006 | LAN / Tailscale |
| CT100 — Network Services | 192.168.1.100 | LAN / Tailscale |
| CT101 — Core Services | 192.168.1.101 | LAN / Tailscale |

| Service | Direct Address | Proxy URL |
|---|---|---|
| Nginx Proxy Manager admin | 192.168.1.100:81 | — |
| AdGuard Home admin | 192.168.1.100:3000 | — |
| Portainer | 192.168.1.101:9000 | portainer.yourdomain.com |
| Vaultwarden | 192.168.1.101:8080 | vault.yourdomain.com |
| Immich | 192.168.1.101:2283 | immich.yourdomain.com |
| Stirling-PDF | 192.168.1.101:8081 | pdf.yourdomain.com |
| Uptime Kuma | 192.168.1.101:3001 | status.yourdomain.com |

---

## Important Notes

- **Backups are non-negotiable.** Vaultwarden data lives on a single NVMe with no redundancy. Backblaze B2 daily backup must be running at all times. Verify it in Portainer after any restart.
- **NVMe-only risks.** There is no ZFS mirror, no RAID, no HDD fallback. A drive failure means total data loss. Plan HDD migration when budget allows.
- **Prometheus retention** is capped at 30 days to prevent the NVMe from filling up. Do not raise this limit without adding storage first.
- **Tailscale key expiry** must be disabled for the Proxmox subnet router node. If the key expires, remote access drops. Check Tailscale admin → machine settings → disable key expiry.
