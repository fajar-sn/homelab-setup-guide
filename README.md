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
        │       ├── AdGuard Home  — systemd service, port 53 / admin :3000
        │       └── Traefik v3    — Docker, ports 80, 443 / dashboard :8080
        │
        └── LXC CT101: Core Services (192.168.1.101)
                └── Docker
                        ├── Portainer CE   — :9000 / :9443
                        ├── Vaultwarden    — :8080  → vault.yourdomain.com
                        ├── Immich         — :2283  → immich.yourdomain.com
                        ├── Stirling-PDF   — :8081  → pdf.yourdomain.com
                        ├── Uptime Kuma    — :3001  → status.yourdomain.com
                        ├── Syncthing      — :8384 (LAN only)
                        └── Memos          — :5230  → memos.yourdomain.com
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
pfSense DNS: vault.yourdomain.com → 192.168.1.100 (Traefik)
    ▼
Traefik reads services.yml → forwards to CT101 Docker service
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
| Network Services | CT100 | 192.168.1.100 | 1 GB | 20 GB | AdGuard Home (systemd) + Traefik (Docker) |
| Core Services | CT101 | 192.168.1.101 | 4 GB | 50 GB | Docker workloads (all user-facing services) |

---

## Tech Stack

### Infrastructure

| Layer | Technology | Role |
|---|---|---|
| Hypervisor | **Proxmox VE** | Runs LXC containers; web UI at `:8006` |
| Containers | **LXC** (Proxmox) | Lightweight OS-level isolation per service group |
| Container runtime | **Docker + Compose** | Runs all user-facing services inside CT101 |
| Container UI | **Portainer CE** | Web UI for managing Docker containers |

### Networking & DNS

| Layer | Technology | Role |
|---|---|---|
| Router / firewall | **pfSense** | DHCP, firewall, split-DNS for `*.yourdomain.com` |
| Primary DNS | **pfBlockerNG** (on pfSense) | Ad-blocking + malware DNS filter |
| Backup DNS | **AdGuard Home** (CT100, systemd) | Fallback DNS if pfSense/pfBlockerNG has issues |
| Reverse proxy | **Traefik v3** (CT100, Docker) | TLS termination, routes by hostname to CT101 services |
| TLS certificates | **Let's Encrypt** (Cloudflare DNS challenge) | Wildcard cert for `*.yourdomain.com` |
| Remote access | **Tailscale** | WireGuard mesh — CGNAT-safe, no open ports |
| Public DNS | **Cloudflare** | Authoritative DNS for `yourdomain.com` |

### Core Services (CT101)

| Service | Technology | Purpose |
|---|---|---|
| Password manager | **Vaultwarden** | Self-hosted Bitwarden-compatible server |
| Photo management | **Immich** | Google Photos replacement; Intel QuickSync ML |
| PDF tools | **Stirling-PDF** | Merge, split, convert PDFs |
| Uptime monitor | **Uptime Kuma** | Service health checks with Telegram alerts |
| File sync | **Syncthing** | P2P encrypted sync: phone/laptop → server (Google Drive replacement) |
| Quick notes | **Memos** | Google Keep replacement; lightweight, Docker-native |

### Backup & Storage

| Tool | Role |
|---|---|
| **restic** | Encrypted, deduplicated backup snapshots |
| **Backblaze B2** | Offsite object storage (Object Lock enabled — ransomware-safe) |
| `vzdump` (Proxmox) | Weekly LXC snapshots to local NVMe |

### Planned (future phases)

| Technology | Phase | Purpose |
|---|---|---|
| Jenkins | Phase 5 | Legacy CI/CD — Groovy DSL, enterprise patterns |
| GitHub Actions (self-hosted runner) | Phase 5 | Modern CI/CD — zero RAM idle |
| SonarQube | Phase 5 | Static code analysis |
| Terraform | Phase 5 | IaC for cloud staging environments |
| Ansible | Phase 5 | Configuration management |
| Prometheus + Grafana | Phase 6 | Metrics, dashboards, Telegram alerts (30-day retention) |
| TrueNAS Scale VM | After HDD | ZFS pool, SMB/NFS shares — blocked until HDD added |
| Home Assistant (HaOS VM) | After 32 GB RAM | Home automation — blocked until RAM upgrade |

---

## Setup Phases

| Phase | Description | Guide | Status |
|---|---|---|---|
| **Phase 0** | Preparation — domain → Cloudflare, BIOS VT-x, download ISO, flash USB | [phase-0-preparation.md](docs/phase-0-preparation.md) | ✅ Written |
| **Phase 1** | Proxmox Installation — install, network bridge, updates, storage | [phase-1-proxmox-installation.md](docs/phase-1-proxmox-installation.md) | ✅ Written |
| **Phase 2** | Network Services — LXC containers, AdGuard Home (systemd), Traefik (Docker), pfSense split-DNS | [phase-2-network-services.md](docs/phase-2-network-services.md) | ✅ Written |
| **Phase 3** | Core Services — Docker, Portainer, Vaultwarden, Immich, Stirling-PDF, Uptime Kuma, Syncthing, Memos | [phase-3-core-services.md](docs/phase-3-core-services.md) | ✅ Written |
| **Phase 4** | Remote Access — Tailscale subnet router, split-DNS, access from anywhere | [phase-4-remote-access.md](docs/phase-4-remote-access.md) | ✅ Written |
| **Phase 5** | Dev & CI/CD — Jenkins, SonarQube, GitHub webhooks, Terraform, Ansible | *(not yet written)* | 🔲 Planned |
| **Phase 6** | Monitoring — Prometheus, Grafana, node_exporter, cAdvisor, Telegram alerts | *(not yet written)* | 🔲 Planned |
| **Phase 7** | Hardening — 2FA everywhere, SSH key-only, DNSSEC, B2 restore drill | *(not yet written)* | 🔲 Planned |
| **Phase 8** | Cloud Staging — Terraform AWS/GCP, Ansible deploy.yml, Jenkins pipeline gate | *(not yet written)* | 🔲 Planned |
| **Phase 9** | VPS Migration — Hetzner VPS, WireGuard + Caddy, replace Tailscale | *(not yet written)* | 🔲 Planned |

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
| Traefik dashboard | http://192.168.1.100:8080 |
| AdGuard Home | http://192.168.1.100:3000 |
| Portainer | http://portainer.yourdomain.com or http://192.168.1.101:9000 |
| Vaultwarden | https://vault.yourdomain.com |
| Immich | https://immich.yourdomain.com |
| Stirling-PDF | https://pdf.yourdomain.com |
| Uptime Kuma | https://status.yourdomain.com |
| Syncthing | http://192.168.1.101:8384 (LAN only) |
| Memos | https://memos.yourdomain.com |

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

All containers (`portainer`, `vaultwarden`, `immich_server`, `immich_postgres`, `immich_redis`, `stirling-pdf`, `uptime-kuma`, `syncthing`, `memos`) should show `Up`. If Docker itself failed to start:

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

Expected: resolves to `192.168.1.100` (Traefik).

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
| Traefik dashboard | 192.168.1.100:8080 | — (LAN only) |
| AdGuard Home admin | 192.168.1.100:3000 | — (LAN only) |
| Portainer | 192.168.1.101:9000 | portainer.yourdomain.com |
| Vaultwarden | 192.168.1.101:8080 | vault.yourdomain.com |
| Immich | 192.168.1.101:2283 | immich.yourdomain.com |
| Stirling-PDF | 192.168.1.101:8081 | pdf.yourdomain.com |
| Uptime Kuma | 192.168.1.101:3001 | status.yourdomain.com |
| Syncthing | 192.168.1.101:8384 | — (LAN only) |
| Memos | 192.168.1.101:5230 | memos.yourdomain.com |

---

## Secrets & Environment Files

All secrets are stored in `.env` files on the server, never in committed files. Each `.env` has a corresponding `.env.example` that is safe to commit and documents the required variables.

| File (on server) | Example template | Phase | Variables |
|---|---|---|---|
| `/opt/traefik/.env` | `/opt/traefik/.env.example` | Phase 2 | `CLOUDFLARE_API_TOKEN` |
| `/opt/vaultwarden/.env` | `/opt/vaultwarden/.env.example` | Phase 3 | `ADMIN_TOKEN` (argon2id hash) |
| `/etc/restic-b2.env` | `/opt/vaultwarden/restic-b2.env.example` | Phase 3 | `B2_ACCOUNT_ID`, `B2_ACCOUNT_KEY`, `RESTIC_PASSWORD` |

**Rules that apply to all `.env` files:**
- `chmod 600` — readable only by root
- Listed in `.gitignore` — never committed
- Actual values stored in Vaultwarden as secure notes (once Vaultwarden is up)

**To set up on a new machine from scratch:**
```bash
cp /opt/traefik/.env.example /opt/traefik/.env
# Fill in real values, then:
chmod 600 /opt/traefik/.env
```

---

## Important Notes

- **Backups are non-negotiable.** Vaultwarden data lives on a single NVMe with no redundancy. Backblaze B2 daily backup must be running at all times. Verify it in Portainer after any restart.
- **NVMe-only risks.** There is no ZFS mirror, no RAID, no HDD fallback. A drive failure means total data loss. Plan HDD migration when budget allows.
- **Prometheus retention** is capped at 30 days to prevent the NVMe from filling up. Do not raise this limit without adding storage first.
- **Tailscale key expiry** must be disabled for the Proxmox subnet router node. If the key expires, remote access drops. Check Tailscale admin → machine settings → disable key expiry.

---

## Planned Additions

For the full roadmap checklist, see [docs/homelab-setup-guide.md § Full Roadmap](docs/homelab-setup-guide.md#12-full-roadmap).

### HDD Migration Plan

Currently running **NVMe-only** — single point of failure, no redundancy, limited storage. Full migration plan: [docs/homelab-setup-guide.md § HDD Migration Plan](docs/homelab-setup-guide.md#13-hdd-migration-plan).

**What is blocked until HDD arrives:**
- Full Immich photo library import (new photos only for now)
- Prometheus retention beyond 30 days
- TrueNAS Scale VM (ZFS, SMB/NFS shares)
- RAM upgrade to 32 GB (required before TrueNAS VM)

**Migration stages:**

| Stage | Trigger | What it unlocks |
|---|---|---|
| **Stage 0** ← current | NVMe only | B2 backup mandatory; constrained storage |
| **Stage A** | 1st 4TB HDD + 32 GB RAM | TrueNAS VM, single-disk ZFS, data off NVMe, full Immich library |
| **Stage B** | 2nd identical 4TB HDD | ZFS mirror (real redundancy), ZFS snapshots, relax B2 frequency |

**Hardware to buy (in order, when budget allows):**
1. 32 GB DDR4-2666 UDIMM kit — mandatory before TrueNAS VM
2. WD Red Plus 4TB CMR — note the exact SKU; Stage B requires an identical drive
3. PCIe SATA x1 card
4. External 2-bay SATA enclosure — buy 2-bay now even with one drive

---

### Centralized Docker Management (Portainer CE + Agents)

Full plan: [docs/plans/2026-04-26-13-22-plan-centralized-docker-management.md](docs/plans/2026-04-26-13-22-plan-centralized-docker-management.md)

**Goal:** One Portainer Server in CT101 manages Docker across all LXC containers via lightweight Portainer Agents — single UI for deploying, updating, and monitoring containers on CT100, CT101, and any future LXC.

**Current state:** Portainer CE in CT101 manages CT101's own containers only.

**To implement (deferred until Phase 5 creates CT102):**
1. Deploy `portainer/agent` container in CT100 on port `9001`
2. When CT102 (dev/CI-CD) exists: deploy agent there too
3. In Portainer UI → **Environments → Add environment → Agent** — register each container's IP + port `9001`

**Why deferred:** CT102 doesn't exist yet. Setting up multi-container management before the full topology is stable adds unnecessary churn. Best done once after Phase 5.
