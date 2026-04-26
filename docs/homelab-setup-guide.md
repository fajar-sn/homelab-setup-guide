# Home Server & Remote Access Setup Guide
**Based on your situation: IndiHome CGNAT + HP EliteDesk 800 G4 SFF + pfSense**
**Stack: DevOps-grade homelab with CI/CD, monitoring, and cloud staging**
**Current phase: NVMe-only (no HDD) — starter configuration**

---

## Table of Contents
1. [Your Network Situation](#1-your-network-situation)
2. [CGNAT Explained](#2-cgnat-explained)
3. [Remote Access Solutions](#3-remote-access-solutions)
4. [Recommended Architecture](#4-recommended-architecture)
5. [Your Hardware: HP EliteDesk 800 G4 SFF](#5-your-hardware-hp-elitedesk-800-g4-sff)
6. [Storage Planning](#6-storage-planning)
7. [OS & Software Stack](#7-os--software-stack)
8. [App Stack — Full Decision Guide](#8-app-stack--full-decision-guide)
9. [CI/CD Pipeline Architecture](#9-cicd-pipeline-architecture)
10. [Cloud Staging: AWS/GCP with Terraform & Ansible](#10-cloud-staging-awsgcp-with-terraform--ansible)
11. [Monitoring Stack](#11-monitoring-stack)
12. [Full Roadmap](#12-full-roadmap)
13. [HDD Migration Plan](#13-hdd-migration-plan)
14. [Buying Guide (Indonesian E-Commerce)](#14-buying-guide-indonesian-e-commerce)

---

## ⚠️ NVMe-Only Compromises

> Read this section first. These are the trade-offs you accept by running without HDD.

```
┌─────────────────────────────────────────────────────────────────────┐
│               COMPROMISES — NVMe Only (No HDD)                     │
├─────────────────────────┬───────────────────────────────────────────┤
│ Compromise              │ Impact & Mitigation                       │
├─────────────────────────┼───────────────────────────────────────────┤
│ No data redundancy      │ If NVMe dies → ALL data lost              │
│                         │ Mitigation: offsite backup is MANDATORY   │
├─────────────────────────┼───────────────────────────────────────────┤
│ Limited capacity        │ 256GB NVMe = ~150GB usable after OS       │
│                         │ 500GB NVMe = ~380GB usable after OS       │
│                         │ Immich library size is constrained        │
├─────────────────────────┼───────────────────────────────────────────┤
│ No ZFS mirror           │ No automatic drive failure protection     │
│                         │ No ZFS snapshots (Proxmox snapshots only) │
├─────────────────────────┼───────────────────────────────────────────┤
│ No TrueNAS VM           │ No SMB/NFS shares to Windows/Mac natively │
│                         │ Mitigation: skip for now, add with HDD    │
├─────────────────────────┼───────────────────────────────────────────┤
│ No NAS file sharing     │ Cannot browse files from Windows Explorer │
│                         │ or Mac Finder over the network            │
├─────────────────────────┼───────────────────────────────────────────┤
│ Vaultwarden risk ↑↑     │ Passwords on single unmirrored drive      │
│                         │ Mitigation: Backblaze B2 backup DAILY     │
│                         │ This is non-negotiable                    │
├─────────────────────────┼───────────────────────────────────────────┤
│ SonarQube history       │ Analysis history stored on NVMe only      │
│                         │ Lost if drive fails before HDD added      │
├─────────────────────────┼───────────────────────────────────────────┤
│ Jenkins artifacts       │ Build artifacts + job history on NVMe     │
│                         │ Regenerable — low risk                    │
├─────────────────────────┼───────────────────────────────────────────┤
│ Prometheus metrics      │ Limited retention (set max 30 days)       │
│                         │ to prevent NVMe from filling up           │
└─────────────────────────┴───────────────────────────────────────────┘
```

### What You GAIN Without HDD (Upside)

```
┌─────────────────────────────────────────────────────────────────┐
│ Benefit                │ Detail                                  │
├────────────────────────┼─────────────────────────────────────────┤
│ Save ~Rp 2,000,000+    │ Skip 2x 4TB HDD + enclosure + SATA card │
│ No TrueNAS VM needed   │ Saves 8GB RAM → 16GB may be enough now  │
│ Simpler architecture   │ No PCIe passthrough, no ZFS pool setup  │
│ Faster to start        │ Day 1 ready — no waiting for hardware   │
│ Learn the stack first  │ Master Proxmox + services before adding │
│                        │ storage complexity                      │
└────────────────────────┴─────────────────────────────────────────┘
```

### RAM Situation Without TrueNAS

```
┌─────────────────────────────────────────┬──────────┐
│  Component                              │  RAM     │
├─────────────────────────────────────────┼──────────┤
│  Proxmox OS                             │   2 GB   │
│  ~~VM: TrueNAS Scale~~ (skipped)        │   0 GB   │
│  LXC: Core Services                     │   4 GB   │
│  LXC: Dev & CI/CD  (SonarQube = 4GB)   │   6 GB   │
│  LXC: Monitoring                        │   2 GB   │
│  LXC: Network Services                  │   1 GB   │
│  LXC: Tunnel/Access                     │   0.5 GB │
│  Buffer / overhead                      │   0.5 GB │
├─────────────────────────────────────────┼──────────┤
│  TOTAL                                  │  16 GB ✅ │
└─────────────────────────────────────────┴──────────┘

✅ 16GB is now workable — no TrueNAS VM frees 8GB.
⚠️ You CANNOT run all LXCs simultaneously at full load.
   Run Dev LXC and Monitoring LXC on-demand, not 24/7.
   When HDD arrives and TrueNAS is added → upgrade to 32GB.
```

---

## 1. Your Network Situation

### Current Topology

```
┌─────────────────────────────────────────────┐
│                  IndiHome ISP               │
│         CGNAT — shared public IP            │
└──────────────────────┬──────────────────────┘
                       │ WAN (100.x.x.x — CGNAT)
                       ▼
            ┌─────────────────────┐
            │  Huawei HG8145V5    │
            │  (ONT / Router)     │
            │  192.168.100.1      │
            └──────────┬──────────┘
                       │ LAN (192.168.100.x)
                       ▼
            ┌─────────────────────┐
            │  pfSense            │  ← WAN: 192.168.100.x (from Huawei)
            │  (Firewall/Router)  │  ← LAN: 192.168.1.1
            │  Double NAT ⚠️      │
            └──────────┬──────────┘
                       │ LAN (192.168.1.x)
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     Your Laptop   Your Phone   Home Server
     192.168.1.x  192.168.1.x  192.168.1.10
```

### Problem: Double NAT + CGNAT
- IndiHome's WAN IP on your Huawei shows `100.x.x.x` → **you are behind CGNAT**
- CGNAT = Carrier-Grade NAT = IndiHome shares one public IP across hundreds of customers
- **You cannot port forward through CGNAT** — inbound connections from the internet are blocked

### How to Verify
| Check | Where | Expected |
|---|---|---|
| Huawei WAN IP | Login `192.168.100.1` → Status → WAN | Shows your ISP-assigned IP |
| Real public IP | whatismyip.com | Should match Huawei WAN IP |
| CGNAT confirmed | Huawei WAN starts with `100.x.x.x` | You are behind CGNAT ❌ |

### If You Want a Public IP
Call IndiHome **147**:
> *"Saya minta public IP / IP publik untuk koneksi saya"*

---

## 2. CGNAT Explained

### What CGNAT Does

```
┌─────────────────────────────────────────────────────────────────┐
│                    IndiHome Infrastructure                      │
│                                                                 │
│  You       (100.x.x.x) ──┐                                     │
│  Neighbor1 (100.x.x.x) ──┤──► IndiHome NAT ──► ONE Public IP  │
│  Neighbor2 (100.x.x.x) ──┤                     (shared)        │
│  Neighbor3 (100.x.x.x) ──┘                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Security Trade-offs
| | Benefit | Explanation |
|---|---|---|
| ✅ | Natural firewall | Nobody can initiate inbound connections to you |
| ✅ | Invisible to scanners | Shodan/bots cannot find your router |
| ❌ | No self-hosting | Cannot expose services to internet directly |
| ❌ | Loss of sovereignty | Must rely on tunnel services |

### The Core Paradox
> **CGNAT makes you more secure against threats you didn't ask for, but removes the security you actively want to build yourself.**

---

## 3. Remote Access Solutions

### Why Tunnels Work Behind CGNAT

```
CGNAT Rule:
  ❌  Internet ──────────────────► You    (blocked — no door in)
  ✅  You      ──────────────────► Internet (always allowed out)

The Exploit — 3 steps:

  STEP 1: Your device dials OUT first (CGNAT allows outbound)
  ┌──────────┐                          ┌─────────────────┐
  │ pfSense  │ ──── outbound conn ────► │ Relay Server    │
  └──────────┘                          │ (Tailscale/VPS) │
                                        └─────────────────┘

  STEP 2: Connection stays open permanently (keepalive)
  ┌──────────┐ ◄──── persistent tunnel ──► ┌─────────────────┐
  │ pfSense  │                              │ Relay Server    │
  └──────────┘                              └─────────────────┘

  STEP 3: Remote traffic rides the existing pipe back in
  ┌───────────┐        ┌─────────────────┐        ┌──────────┐
  │Your Phone │ ──────►│ Relay Server    │ ──────►│ pfSense  │
  └───────────┘        └─────────────────┘        └──────────┘
                       CGNAT never sees a NEW inbound connection ✅
```

---

### Option A: Tailscale (Free, Easiest)

```
  ┌────────────────┐         ┌──────────────────────┐
  │ Your Phone /   │         │ Tailscale             │
  │ Laptop         │◄───────►│ Coordination Server   │
  └────────────────┘         │ (brokers connection)  │
                             └──────────┬───────────┘
                                        │ P2P WireGuard
                                        ▼
                             ┌──────────────────────┐
                             │ pfSense              │
                             │ (Tailscale installed)│
                             └──────────┬───────────┘
                                        │
                                        ▼
                                  Your LAN + Server
```

| | Detail |
|---|---|
| **Protocol** | WireGuard-based |
| **Free tier** | 3 users, 100 devices |
| **Best for** | Private access — services, SSH, pfSense admin |

---

### Option B: Cloudflare Tunnel (Free, Public URLs)

```
  ┌───────────────┐        ┌──────────────────┐
  │ Anyone on     │        │ Cloudflare Edge  │
  │ Internet      │───────►│ (your domain     │
  └───────────────┘        │  points here)    │
                           └────────┬─────────┘
                                    │ pushes down existing pipe
                                    ▼
                           ┌──────────────────┐
                           │ cloudflared LXC  │◄── dials OUT to Cloudflare
                           │ (inside your LAN)│
                           └────────┬─────────┘
                                    │
                                    ▼
                           Nginx Proxy Manager
                           → your services
```

| | Detail |
|---|---|
| **Best for** | Public URLs — dev/staging environments |
| **Limitation** | All traffic through Cloudflare, no large media |

---

### Option C: VPS WireGuard Relay (Paid, Best Long-Term)

```
  ┌───────────────┐        ┌──────────────────────────┐
  │ Your Phone /  │        │ Hetzner VPS              │
  │ Anyone        │───────►│ Public IP                │
  └───────────────┘        │ ┌──────────┐ ┌─────────┐ │
                           │ │  Caddy   │ │WireGuard│ │
                           │ │ (SSL)    │ │ Server  │ │
                           │ └──────────┘ └────┬────┘ │
                           └──────────────────┼───────┘
                                              │ WireGuard tunnel
                                              ▼
                                    ┌──────────────────┐
                                    │ pfSense          │
                                    │ WireGuard client │
                                    └────────┬─────────┘
                                             ▼
                                       Your LAN + Server
```

---

### Full Comparison

| Factor | Tailscale | Cloudflare Tunnel | VPS WireGuard |
|---|---|---|---|
| **Cost** | Free | Free | ~$5/month |
| **Works behind CGNAT** | ✅ | ✅ | ✅ |
| **OpenVPN support** | ❌ | ❌ | ✅ Restored |
| **Custom domain** | ❌ | ✅ | ✅ |
| **Traffic privacy** | ⚠️ | ⚠️ Cloudflare sees it | ✅ E2E |
| **Long-term viability** | ⚠️ | ⚠️ | ✅ |

---

## 4. Recommended Architecture

### Full Network Topology

```
                               INTERNET
                                  │
            ┌─────────────────────┼──────────────────────┐
            ▼                     ▼                       ▼
 ┌────────────────────┐  ┌─────────────────┐  ┌─────────────────────┐
 │   Hetzner VPS      │  │    Tailscale    │  │    AWS / GCP        │
 │   (Singapore)      │  │    Network      │  │    Cloud Staging    │
 │   ~€3.50/month     │  │    P2P mesh     │  │  Provisioned by     │
 │  ┌──────────────┐  │  └────────┬────────┘  │  Terraform +        │
 │  │ Caddy (SSL)  │  │           │            │  Ansible            │
 │  └──────┬───────┘  │           │            └─────────────────────┘
 │  ┌──────▼───────┐  │           │
 │  │  WireGuard   │  │           │
 │  │  Server      │  │           │
 │  └──────┬───────┘  │           │
 └─────────┼──────────┘           │
           │ WireGuard             │ WireGuard P2P
           │ (always-on tunnel)    │ (outbound from pfSense)
           ▼                       ▼
┌──────────────────────────────────────────────────────────────┐
│                      YOUR HOME NETWORK                       │
│                                                              │
│   ┌──────────────────────────────────────────────────────┐  │
│   │  Huawei HG8145V5 ◄── IndiHome CGNAT (100.x.x.x)     │  │
│   └────────────────────────┬─────────────────────────────┘  │
│                            │                                 │
│   ┌────────────────────────▼─────────────────────────────┐  │
│   │  pfSense (192.168.1.1)                               │  │
│   │  ┌──────────────┐  ┌─────────────┐  ┌────────────┐  │  │
│   │  │ DHCP Server  │  │  OpenVPN    │  │ WireGuard  │  │  │
│   │  │ → AdGuard IP │  │  Server     │  │ Client     │  │  │
│   │  └──────────────┘  │  (Ph.2 ✅)  │  │ → VPS      │  │  │
│   │                     └─────────────┘  └────────────┘  │  │
│   └─────────────────────────┬────────────────────────────┘  │
│                             │ LAN 192.168.1.x                │
│          ┌──────────────────┼──────────────────┐             │
│          ▼                  ▼                  ▼             │
│     Your Laptop        Your Phone         Wife's Phone       │
│                                                              │
│                             │                                │
│                             ▼                                │
│              HP EliteDesk 800 G4 SFF                        │
│                    192.168.1.10                              │
└──────────────────────────────────────────────────────────────┘
```

---

### Server Internal Architecture (NVMe-Only)

```
┌─────────────────────────────────────────────────────────────────────────┐
│               HP EliteDesk 800 G4 SFF                                   │
│       i5-8500 (6C/6T 3.0GHz) │ 16GB DDR4-2666 │ Intel UHD 630         │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                       PROXMOX VE                                  │  │
│  │                  192.168.1.10 : 8006                              │  │
│  │                                                                   │  │
│  │  ┌─────────────────────────────────────────────────────────────┐ │  │
│  │  │  PHYSICAL STORAGE (NVMe only — no HDD)                     │ │  │
│  │  │                                                             │ │  │
│  │  │  ┌────────────────────────────────────────────────────┐    │ │  │
│  │  │  │ M.2 NVMe SSD (500GB recommended)                   │    │ │  │
│  │  │  │                                                     │    │ │  │
│  │  │  │ Proxmox OS:        32 GB                           │    │ │  │
│  │  │  │ LXC rootfs (all):  ~100 GB                         │    │ │  │
│  │  │  │ Service data:      ~200 GB                         │    │ │  │
│  │  │  │ Proxmox backups:   ~100 GB                         │    │ │  │
│  │  │  │                   ──────────                        │    │ │  │
│  │  │  │ Total:            ~432 GB  ← fits in 500GB NVMe    │    │ │  │
│  │  │  └────────────────────────────────────────────────────┘    │ │  │
│  │  │                                                             │ │  │
│  │  │  ~~TrueNAS VM~~ → SKIPPED (no HDD = no ZFS pool)          │ │  │
│  │  └─────────────────────────────────────────────────────────────┘ │  │
│  │                                                                   │  │
│  │  ┌───────────────────────┐  ┌────────────────────────────────┐   │  │
│  │  │  LXC: Core Services   │  │  LXC: Dev & CI/CD              │   │  │
│  │  │  RAM: 4GB [Docker]    │  │  RAM: 6GB [Docker]             │   │  │
│  │  │  ┌─────────────────┐  │  │  ┌──────────────────────────┐  │   │  │
│  │  │  │ Immich   :2283  │  │  │  │ GitHub (remote) :webhook │  │   │  │
│  │  │  │ Vaultwarden     │  │  │  │ Jenkins         :8080    │  │   │  │
│  │  │  │          :8080  │  │  │  │ SonarQube       :9000    │  │   │  │
│  │  │  │ Stirling-PDF    │  │  │  │  (4GB JVM heap)          │  │   │  │
│  │  │  │          :8081  │  │  │  └──────────────────────────┘  │   │  │
│  │  │  │ Uptime Kuma     │  │  │  Data on: /var/lib/docker/     │   │  │
│  │  │  │          :3001  │  │  │  (stored on NVMe directly)     │   │  │
│  │  │  │ Portainer:9000  │  │  └────────────────────────────────┘   │  │
│  │  │  └─────────────────┘  │                                       │  │
│  │  │  Data on: NVMe only   │  ┌────────────────────────────────┐   │  │
│  │  │  ⚠️ backup offsite    │  │  LXC: Monitoring               │   │  │
│  │  └───────────────────────┘  │  RAM: 2GB [Docker]             │   │  │
│  │                             │  ┌──────────────────────────┐  │   │  │
│  │                             │  │ Prometheus (30d retention)│  │   │  │
│  │                             │  │ node_exporter    :9100   │  │   │  │
│  │                             │  │ cAdvisor         :8080   │  │   │  │
│  │                             │  │ Grafana          :3000   │  │   │  │
│  │                             │  └──────────────────────────┘  │   │  │
│  │                             │  ⚠️ limit Prometheus retention  │   │  │
│  │                             │     to 30 days (disk guard)    │   │  │
│  │                             └────────────────────────────────┘   │  │
│  │                                                                   │  │
│  │  ┌───────────────────────────────┐  ┌───────────────────────┐    │  │
│  │  │  LXC: Network Services        │  │  LXC: Tunnel/Access   │    │  │
│  │  │  RAM: 1GB  [Mixed]            │  │  RAM: 0.5GB           │    │  │
│  │  │  ┌───────────────────────┐    │  │  Tailscale router     │    │  │
│  │  │  │ Nginx Proxy Manager   │    │  │  → WireGuard (Ph.2)   │    │  │
│  │  │  │ cloudflared (Phase 1) │    │  └───────────────────────┘    │  │
│  │  │  └───────────────────────┘    │                               │  │
│  │  │  DNS: pfBlockerNG (pfSense)   │                               │  │
│  │  └───────────────────────────────┘                               │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### Storage Layout on NVMe (500GB)

```
┌─────────────────────────────────────────────────────────┐
│  M.2 NVMe 500GB — Full Allocation                       │
├───────────────────────────────────┬─────────────────────┤
│  Proxmox OS + system              │  32 GB              │
├───────────────────────────────────┼─────────────────────┤
│  LXC: Core Services rootfs        │  20 GB              │
│    └── Immich library             │  100 GB  ⚠️ limited │
│    └── Vaultwarden data           │  2 GB               │
│    └── Stirling-PDF temp          │  5 GB               │
├───────────────────────────────────┼─────────────────────┤
│  LXC: Dev & CI/CD rootfs          │  20 GB              │
│    └── SonarQube data + DB        │  30 GB              │
│    └── Jenkins home               │  30 GB              │
├───────────────────────────────────┼─────────────────────┤
│  LXC: Monitoring rootfs           │  10 GB              │
│    └── Prometheus TSDB (30d max)  │  30 GB              │
├───────────────────────────────────┼─────────────────────┤
│  LXC: Network Services rootfs     │  10 GB              │
│  LXC: Tunnel/Access rootfs        │  5 GB               │
├───────────────────────────────────┼─────────────────────┤
│  Proxmox backup storage           │  80 GB              │
├───────────────────────────────────┼─────────────────────┤
│  Free headroom                    │  ~106 GB            │
├───────────────────────────────────┼─────────────────────┤
│  TOTAL                            │  500 GB ✅          │
└───────────────────────────────────┴─────────────────────┘

⚠️  Immich limited to ~100GB without HDD
    = roughly 2–3 years of casual photos for 2 people
    = NOT enough for a full photo library migration
    → Use Immich in NVMe phase for NEW photos only
    → Migrate old library when HDD arrives
```

---

### Phase 1 → Phase 2 Traffic Flow

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PHASE 1 (Free — Tailscale + Cloudflare)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Private access (Tailscale):
┌────────────┐     ┌──────────────────┐     ┌──────────────┐
│ Your Phone │────►│ Tailscale relay  │────►│ pfSense      │
│ (anywhere) │     │ (or P2P direct)  │     │ → LAN 1.x    │
└────────────┘     └──────────────────┘     └──────┬───────┘
                                                   │
                           ┌───────────────────────┼──────────┐
                           ▼                       ▼          ▼
                        Immich               Vaultwarden  Proxmox
                        (photos)             (passwords)  :8006

Public URL access (Cloudflare Tunnel):
┌────────────┐     ┌──────────────────┐     ┌──────────────────────┐
│  Browser   │────►│ Cloudflare Edge  │────►│ cloudflared LXC      │
│ (anyone)   │     │ dev.domain.com   │     │ → Nginx Proxy Mgr    │
└────────────┘     └──────────────────┘     │   → dev service      │
                                            │   → staging service  │
                                            └──────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PHASE 2 (Sovereign VPS — WireGuard + Caddy)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌────────────┐     ┌────────────────────────┐
│  Browser / │────►│  Hetzner VPS           │
│  Phone     │     │  yourdomain.com        │
└────────────┘     │  ┌─────────────────┐   │
                   │  │ Caddy (Auto SSL)│   │
                   │  └────────┬────────┘   │
                   │  ┌────────▼────────┐   │
                   │  │ WireGuard Server│   │
                   │  └────────┬────────┘   │
                   └───────────┼────────────┘
                               │ encrypted tunnel
                               ▼
                   ┌───────────────────────┐
                   │ pfSense               │
                   │ OpenVPN server ✅     │
                   └───────────┬───────────┘
                               ▼
                   Nginx Proxy Manager → services
```

---

## 5. Your Hardware: HP EliteDesk 800 G4 SFF

*Based on official HP Maintenance and Service Guide (c06472102)*

### Physical Layout

```
┌──────────────────────────────────────────────────────┐
│           HP EliteDesk 800 G4 SFF — Inside           │
│                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ CPU         │  │ RAM Slots    │  │ M.2 2280   │  │
│  │ i5-8500     │  │ 2x UDIMM     │  │ NVMe SSD   │  │
│  │ 6C/6T       │  │ 16GB now     │  │ ← ALL data │  │
│  │ UHD 630     │  │ 32GB later   │  │   goes here│  │
│  └─────────────┘  └──────────────┘  └────────────┘  │
│                                                      │
│  ┌─────────────────────┐  ┌────────────────────────┐ │
│  │ PCIe x16 slot       │  │ 3.5" Drive Bay         │ │
│  │ (unused)            │  │ ← EMPTY for now        │ │
│  └─────────────────────┘  └────────────────────────┘ │
│  ┌─────────────────────┐  ┌────────────────────────┐ │
│  │ PCIe x1 slot        │  │ 2.5" Drive Cage        │ │
│  │ ← EMPTY for now     │  │ ← EMPTY for now        │ │
│  │ (SATA card later)   │  └────────────────────────┘ │
│  └─────────────────────┘                             │
└──────────────────────────────────────────────────────┘

When HDD arrives (future):
PCIe x1 ← SATA card → External 2-Bay Enclosure
                         ├── WD Red Plus 4TB
                         └── WD Red Plus 4TB
```

### Confirmed Specs
| Component | Spec | Note |
|---|---|---|
| CPU | Intel Core i5-8500, 6C/6T, 3.0GHz | ✅ |
| iGPU | Intel UHD 630 | QuickSync for Immich ✅ |
| RAM | 16GB DDR4-2666 UDIMM | Sufficient for NVMe phase |
| RAM (future) | Upgrade to 32GB when HDD/TrueNAS added | Mandatory then |
| Virtualization | VT-x + VT-d | ✅ |
| PSU | 250W | ✅ |

### Enable VT-x in BIOS
```
Power on → F10 → Advanced → Device Options
→ Enable Intel Virtualization Technology (VT-x)
→ Save and exit
(VT-d not needed yet — no PCIe passthrough without HDD)
```

---

## 6. Storage Planning

### NVMe-Only Storage Rules

```
  ┌────────────────────────────────────────────────────┐
  │  RULES for NVMe-only phase                        │
  │                                                    │
  │  1. Prometheus retention: max 30 days             │
  │     (prevents TSDB from growing unbounded)        │
  │                                                    │
  │  2. Jenkins: set build artifact retention         │
  │     Keep last 10 builds only                      │
  │                                                    │
  │  3. Immich: new photos only                       │
  │     Don't import your full historical library yet │
  │     Wait for HDD                                  │
  │                                                    │
  │  4. Docker image cleanup: weekly                  │
  │     docker system prune -af --volumes             │
  │     (removes unused images + volumes)             │
  │                                                    │
  │  5. Uptime Kuma: monitor NVMe disk usage          │
  │     Alert at 70% — act before it's too late       │
  └────────────────────────────────────────────────────┘
```

### Backup Strategy (Non-Negotiable on NVMe)

```
No ZFS = no snapshots = offsite backup is your ONLY safety net.

┌──────────────────────────────────────────────────────────┐
│  What MUST be backed up offsite (Backblaze B2 free 10GB) │
├──────────────────────────────────────────────────────────┤
│  Priority 1 — Vaultwarden /data/db.sqlite3               │
│               → Rclone sync daily → B2                   │
│               → If lost: all passwords gone forever      │
├──────────────────────────────────────────────────────────┤
│  Priority 2 — Immich /library                            │
│               → Rclone sync weekly → B2                  │
│               → Photos are irreplaceable                 │
├──────────────────────────────────────────────────────────┤
│  Priority 3 — Vaultwarden offline backup (USB)           │
│               → Monthly export: Vaultwarden Admin        │
│                 Panel → Export → encrypted JSON          │
│               → Copy to USB drive → store offline        │
│               → Offline copy protects against:           │
│                 B2 outage, Vaultwarden corruption,       │
│                 accidental cloud delete                  │
├──────────────────────────────────────────────────────────┤
│  Priority 4 — Proxmox LXC backup (vzdump)                │
│               → Weekly to local NVMe /backup dir         │
│               → Allows fast LXC restore                  │
└──────────────────────────────────────────────────────────┘

Backblaze B2 free tier: 10GB storage, 1GB/day egress
Rclone setup: rclone config → B2 → bucket per service
```

---

## 7. OS & Software Stack

### Why Proxmox

| | Ubuntu Server | TrueNAS Scale | Proxmox VE |
|---|---|---|---|
| Virtualization | ❌ Limited | ⚠️ Basic | ✅ Full hypervisor |
| NAS capability | ❌ Manual | ✅ Excellent | ✅ Via TrueNAS VM (later) |
| Dev/CI isolation | ⚠️ Docker only | ❌ Poor | ✅ Separate LXC per role |
| Snapshot/rollback | ❌ | ✅ | ✅ |
| Expandable later | ⚠️ | ⚠️ | ✅ Add TrueNAS VM when HDD arrives |

### Full Software Stack

| Layer | Software | Status |
|---|---|---|
| Hypervisor | Proxmox VE | ✅ Install now |
| Storage VM | ~~TrueNAS Scale~~ | ⏳ Add when HDD arrives |
| Container UI | Portainer | ✅ Install now |
| Photos | Immich | ✅ Install now (new photos only) |
| Password manager | Vaultwarden | ✅ Install now + backup immediately |
| PDF editor | Stirling-PDF | ✅ Install now |
| Uptime monitor | Uptime Kuma | ✅ Install now |
| Git hosting | GitHub (remote) | ✅ Use existing account |
| CI/CD | Jenkins | ✅ Install now |
| Code quality | SonarQube | ✅ Install now |
| IaC | Terraform CLI | ✅ Install now |
| Config mgmt | Ansible CLI | ✅ Install now |
| Metrics | Prometheus + exporters | ✅ Install now (30d retention) |
| Dashboards | Grafana | ✅ Install now |
| DNS Primary | pfBlockerNG (pfSense) | ✅ Already active — no action needed |
| DNS Backup | AdGuard Home (LXC) | ✅ Install as fallback |
| Reverse proxy | Nginx Proxy Manager | ✅ Install now |
| Remote (Phase 1) | Tailscale | ✅ Install now |
| Public URLs | Cloudflare Tunnel | ✅ Install now |
| Remote (Phase 2) | WireGuard on VPS | ⏳ Month 3–6 |
| SMB/NFS shares | ~~TrueNAS~~ | ⏳ Add when HDD arrives |

---

## 8. App Stack — Full Decision Guide

### Final Decisions

| App | Decision | Reasoning |
|---|---|---|
| **Docker** | ✅ Keep | Core runtime |
| **SonarQube** | ✅ Keep | Needs 4GB RAM — run on-demand if memory pressure |
| **Ansible** | ✅ Keep | CLI tool — no RAM cost when idle |
| **Terraform** | ✅ Keep | CLI tool — no RAM cost when idle |
| **Tracetest** | ❌ Skip | Needs full OpenTelemetry stack. Revisit later |
| **Stirling-PDF** | ✅ Keep | Lightweight, Docker-native |
| **Tailscale** | ✅ Phase 1 | Replace with WireGuard in Phase 2 |
| **Vaultwarden** | ✅ Keep | 50MB RAM. **Backup daily — non-negotiable** |
| **Cloudflare Tunnel** | ✅ Phase 1 | Public URLs for dev/staging |
| **Nginx Proxy Manager** | ✅ Keep | Internal reverse proxy |
| **AdGuard Home** | ✅ Add as backup | Secondary DNS — fallback if pfBlockerNG fails |
| **pfBlockerNG** | ✅ Already active | Primary DNS blocking on pfSense |
| **Grafana + Prometheus** | ✅ Keep | 30-day retention on NVMe |
| **Uptime Kuma** | ✅ Keep | Monitor disk usage alert at 70% |
| **Teleport** | ❌ Replace | Tailscale SSH handles this at zero cost |
| **Gitea** | ❌ Skip | Using GitHub instead — saves 20GB disk + RAM |
| **Jenkins** | ✅ Add | CI/CD orchestration |

### DNS Setup — pfBlockerNG Primary + AdGuard Home Backup
```
pfBlockerNG is the PRIMARY DNS filter on pfSense.
AdGuard Home runs in Core Services LXC as a SECONDARY/BACKUP.

DNS chain (normal):
pfSense Unbound  → resolves all client queries
pfBlockerNG      → intercepts and blocks ads + malicious domains
                 → forwards clean queries upstream (1.1.1.1 / 9.9.9.9)

DNS chain (fallback — if pfSense/pfBlockerNG has issues):
Clients → AdGuard Home LXC :3000
        → forwards to 1.1.1.1 / 9.9.9.9
        → still blocks ads via own blocklists

AdGuard Home config:
  Install: Docker in Core Services LXC, port 53 + :3000 UI
  Upstream DNS: 1.1.1.1, 9.9.9.9
  Blocklists: use same lists as pfBlockerNG (OISD, Steven Black)
  ⚠️  Do NOT set AdGuard as default DNS on pfSense —
      only point individual devices to it when pfBlockerNG is down.
      Running both as active resolvers = DNS loops.

Local *.lan overrides → add in pfSense:
  Services → DNS Resolver → Host Overrides
  proxmox.lan  → 192.168.1.10
  vault.lan    → [NPM LXC IP]   (proxies to Vaultwarden)
  immich.lan   → [NPM LXC IP]
  pdf.lan      → [NPM LXC IP]
  status.lan   → [NPM LXC IP]
  adguard.lan  → [Core Services LXC IP]
```

---

## 9. CI/CD Pipeline Architecture

### Full Pipeline Flow

```
  Developer pushes code
         │
         ▼
  ┌──────────────┐
  │ GitHub       │
  │ (remote)     │
  └──────┬───────┘
         │ webhook (push / pull request)
         ▼
  ┌──────────────────────────────────────────────────┐
  │  Jenkins                                         │
  │                                                  │
  │  Stage 1: Checkout                               │
  │  Stage 2: Build                                  │
  │  Stage 3: Test                                   │
  │  Stage 4: SonarQube Scan ──────────────────────┐ │
  │                                                │ │
  │                          ┌─────────────────────┘ │
  │                          ▼                       │
  │                    SonarQube                     │
  │                    Quality Gate                  │
  │                    ┌──────┴──────┐               │
  │                   PASS          FAIL             │
  │                    │             │               │
  │                    ▼             ▼               │
  │  Stage 5: Deploy  ✅      Notify + abort ✋      │
  └──────────┬───────────────────────────────────────┘
             │
      ┌──────┴──────────┐
      ▼                 ▼
  Terraform         Ansible
  apply             deploy.yml
  (provision        (configure
   EC2/GCE)         + deploy app)
      └──────┬──────────┘
             ▼
     staging.yourdomain.com ✅
```

### Jenkinsfile Sample
```groovy
pipeline {
  agent any
  stages {
    stage('Build') {
      steps { sh 'mvn clean package -DskipTests' }
    }
    stage('Test') {
      steps { sh 'mvn test' }
    }
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh 'mvn sonar:sonar'
        }
      }
    }
    stage('Quality Gate') {
      steps {
        timeout(time: 1, unit: 'HOURS') {
          waitForQualityGate abortPipeline: true
        }
      }
    }
    stage('Deploy to Staging') {
      when { branch 'main' }
      steps {
        sh 'ansible-playbook -i inventory/staging deploy.yml'
      }
    }
  }
}
```

---

## 10. Cloud Staging: AWS/GCP with Terraform & Ansible

### IaC Flow

```
  Jenkins pipeline
       │
       ├── terraform apply
       │     └── provisions: EC2/GCE, VPC, SG, DNS record
       │
       └── ansible-playbook deploy.yml
             └── configures: runtime, Docker image, env vars, start app
```

### Terraform Scope
```
  ├── Hetzner VPS    → server, firewall, DNS records (yourdomain.com → VPS IP)
  ├── AWS Staging    → EC2 t3.micro, VPC, SG, Route53
  └── GCP Staging    → GCE e2-micro, VPC firewall, Cloud DNS

  State: Terraform Cloud (free tier)
  ⚠️  Never store state locally — contains secrets
```

### Ansible Scope
```
  ├── Homelab LXCs   → post-creation config, Docker install
  ├── Hetzner VPS    → WireGuard, Caddy, SSH hardening
  └── Staging servers → runtime, Docker image, env vars

  Secrets: ansible-vault (vault password stored in Vaultwarden)
```

---

## 11. Monitoring Stack

### What Gets Monitored

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Prometheus scrape targets                                   │
  │                                                              │
  │  node_exporter  ──► host CPU/RAM/disk/network               │
  │  cAdvisor       ──► per-container metrics                   │
  │  pfSense Telegraf──► network throughput, DNS queries        │
  │                      │                                       │
  │                       └──► all → Prometheus TSDB            │
  │                                   (30 day max on NVMe)      │
  │                                        │                    │
  │                                        ▼                    │
  │                                    Grafana                  │
  │                                    Dashboards:              │
  │                                    ├── Proxmox overview     │
  │                                    ├── Docker containers    │
  │                                    ├── pfSense network      │
  │                                    ├── NVMe disk usage ⚠️   │
  │                                    └── Jenkins CI metrics   │
  └──────────────────────────────────────────────────────────────┘
```

### Critical Alert: Disk Usage

```
⚠️  On NVMe-only setup, disk is your scarcest resource.

Grafana alert rules (mandatory):
  🔴 CRITICAL: NVMe usage > 85% → immediate action
  🟡 WARNING:  NVMe usage > 70% → plan cleanup
  🔴 CRITICAL: Vaultwarden unreachable > 5 min
  🟡 WARNING:  Any service down > 5 min

Notification: Telegram bot (easiest to set up)
```

---

## 12. Full Roadmap

### Phase 0 — Preparation (Day 1)
```
[ ] Set up your domain (bought from RumahWeb):
    Option A — Transfer DNS management to Cloudflare (recommended):
      RumahWeb Client Area → Domains → Manage → Nameservers
      → Change to: lara.ns.cloudflare.com / sid.ns.cloudflare.com
      → Then manage all DNS records from Cloudflare dashboard
      → Benefit: free SSL, Cloudflare Tunnel, DDoS protection, fast CDN
    Option B — Keep DNS on RumahWeb:
      Add DNS records manually via RumahWeb cPanel → Zone Editor
      → Less flexible, cannot use Cloudflare Tunnel without NS transfer

    Domain usage plan:
      yourdomain.com          → main site (future)
      dev.yourdomain.com      → CI/CD staging preview (Cloudflare Tunnel)
      vault.yourdomain.com    → Vaultwarden (Tailscale-only, not public)
      immich.yourdomain.com   → Immich (Tailscale-only, not public)
      status.yourdomain.com   → Uptime Kuma (optional public)
      *.yourdomain.com        → Phase 2: all services via Caddy on Hetzner VPS

    ⚠️  Vaultwarden + Immich should NOT be public-facing via Cloudflare Tunnel.
        Keep them private — accessible only via Tailscale.
        Only dev/staging and public-safe services get public subdomains.

[ ] Enable VT-x in HP EliteDesk BIOS (F10 on boot)
[ ] Download Proxmox VE ISO from proxmox.com
[ ] Flash ISO to USB (Balena Etcher)
[ ] Buy 500GB M.2 NVMe SSD if not owned (see Section 14)
```

### Phase 1 — Proxmox Foundation (Day 1–2)
```
[ ] Install Proxmox VE on M.2 NVMe SSD
[ ] Configure networking (bridge to pfSense LAN)
[ ] Set static IP: 192.168.1.10
[ ] Access Proxmox UI: https://192.168.1.10:8006
[ ] Update Proxmox, configure no-subscription repo
[ ] Configure Proxmox backup dir on NVMe (/var/lib/vz/dump)
```

### Phase 2 — Network Services (Day 2)
```
[ ] Create Network Services LXC (1GB RAM)
[ ] Install Nginx Proxy Manager (Docker)
[ ] Add local DNS overrides in pfSense:
    Services → DNS Resolver → Host Overrides
    → proxmox.lan  → 192.168.1.10
    → vault.lan    → [NPM LXC IP]
    → immich.lan   → [NPM LXC IP]
    → pdf.lan      → [NPM LXC IP]
    → status.lan   → [NPM LXC IP]
    → adguard.lan  → [Core Services LXC IP]
[ ] Point your domain subdomains in Cloudflare DNS:
    → homelab.yourdomain.com  → (Phase 1: Cloudflare Tunnel)
    → dev.yourdomain.com      → CI/CD staging preview
    → vault.yourdomain.com    → Vaultwarden (Tailscale-only, not public)
    Note: pfBlockerNG handles primary DNS blocking on pfSense
```

### Phase 3 — Core Services (Day 2–3)
```
[ ] Create Core Services LXC (4GB RAM)
[ ] Install Docker + Portainer
[ ] Deploy Vaultwarden
[ ] ⚠️ IMMEDIATELY set up Rclone → Backblaze B2 backup for Vaultwarden
[ ] ⚠️ Set up monthly offline backup for Vaultwarden:
    Vaultwarden Admin Panel → Export Vault → Encrypted JSON
    → Copy exported file to USB drive → store securely offline
[ ] Deploy AdGuard Home (backup DNS, port 53 + UI :3000)
[ ] Deploy Immich (new photos only — do NOT import full library yet)
[ ] Deploy Stirling-PDF
[ ] Deploy Uptime Kuma
[ ] Add Uptime Kuma disk usage monitor (alert at 70%)
[ ] Configure Nginx PM: vault.lan, immich.lan, pdf.lan, status.lan, adguard.lan
```

### Phase 4 — Remote Access (Day 3–4)
```
[ ] Install Tailscale on pfSense
[ ] Enable subnet routing (192.168.1.0/24)
[ ] Install Tailscale on your devices + wife's devices
[ ] Test remote access: Vaultwarden + Immich via Tailscale
[ ] Deploy cloudflared in Network Services LXC
[ ] Point dev.yourdomain.com → Nginx PM → dev service
```

### Phase 5 — Dev & CI/CD (Week 2)
```
[ ] Create Dev & CI/CD LXC (6GB RAM)
[ ] Deploy SonarQube + PostgreSQL (Docker Compose)
[ ] Deploy Jenkins
[ ] Configure Jenkins → GitHub integration:
    Jenkins → Manage Jenkins → Plugins → GitHub plugin
    Create GitHub Personal Access Token → add to Jenkins credentials
    Configure webhook on GitHub repo → http://[Jenkins IP]:8080/github-webhook/
[ ] Connect Jenkins → SonarQube (SonarQube Scanner plugin)
[ ] Create first pipeline: build → test → scan → quality gate
[ ] Install Terraform CLI + Ansible CLI
[ ] Test: Terraform plan, Ansible ping
[ ] Set Jenkins artifact retention: keep last 10 builds only
```

### Phase 6 — Monitoring (Week 2)
```
[ ] Create Monitoring LXC (2GB RAM)
[ ] Deploy Prometheus (set --storage.tsdb.retention.time=30d)
[ ] Deploy node_exporter, cAdvisor, Grafana
[ ] Install Telegraf on pfSense
[ ] Import Grafana dashboards (Proxmox, Docker, pfSense)
[ ] Set up Grafana alerts → Telegram bot
[ ] Add NVMe disk usage alert at 70%
```

### Phase 7 — Hardening (Week 3)
```
[ ] Change all default passwords
[ ] Enable 2FA: Proxmox, Jenkins, SonarQube, Vaultwarden
[ ] SSH: key-based auth only
[ ] pfSense DNS Resolver: enable DNSSEC (Services → DNS Resolver → DNSSEC)
    → Unbound + pfBlockerNG support DNSSEC natively — no conflict
[ ] pfSense firewall: restrict unnecessary rules
[ ] Verify Backblaze B2 backup is running (check B2 console)
[ ] Test restore: restore Vaultwarden from B2 backup (dry run)
```

### Phase 8 — Cloud Staging (Month 2)
```
[ ] Create AWS/GCP free tier account
[ ] Write Terraform code for staging environment
[ ] Store state in Terraform Cloud (free)
[ ] Write Ansible deploy.yml
[ ] Add to Jenkins pipeline: deploy to AWS/GCP after quality gate
[ ] Test: staging.yourdomain.com → your app ✅
```

### Phase 9 — VPS Migration (Month 3–6)
```
[ ] Rent Hetzner VPS CAX11 Singapore (~€3.29/month)
[ ] Terraform: provision VPS
[ ] Ansible: install WireGuard + Caddy on VPS
[ ] Replace Tailscale with WireGuard on pfSense
[ ] Replace cloudflared with Caddy on VPS
[ ] Update DNS: yourdomain.com → VPS IP
[ ] OpenVPN on pfSense restored ✅
```

---

## 13. HDD Migration Plan

> You can buy HDDs one at a time. All paths lead to the same final state.
> Follow Stage A when the 1st HDD arrives, Stage B when the 2nd arrives.

### Migration Path Overview

```
Stage 0 (current)    Stage A (1st HDD)         Stage B (2nd HDD)
NVMe only        →   Single-disk ZFS        →   ZFS mirror (full protection)
No TrueNAS           TrueNAS VM added           Mirror completed
16GB RAM OK          32GB RAM REQUIRED          32GB RAM (no change)
B2 backup MUST       B2 backup still MUST       B2 backup still recommended
Immich: new only     Full library now           Full library confirmed
Prometheus: 30d      Prometheus: 90d            Prometheus: 1 year
```

### What Changes at Each Stage

```
┌─────────────────────────────┬──────────────────────────────┬──────────────────────────────┐
│  Stage 0 (NVMe only)        │  Stage A (1st HDD)           │  Stage B (2nd HDD)           │
├─────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
│  No TrueNAS VM              │  TrueNAS Scale VM added      │  No VM change                │
│  Data on NVMe               │  Data migrated to ZFS        │  Data already on ZFS         │
│  No SMB shares              │  SMB shares available        │  SMB shares working          │
│  No ZFS snapshots           │  ZFS checksums only ⚠️        │  ZFS mirror + snapshots ✅   │
│  No redundancy              │  No redundancy yet ⚠️         │  Full redundancy ✅           │
│  B2 backup CRITICAL         │  B2 backup still MUST        │  B2 backup: relax frequency  │
│  16GB RAM OK                │  32GB RAM REQUIRED           │  32GB RAM (no change)        │
│  Immich: new photos only    │  Import full library now     │  Full library confirmed      │
│  Prometheus: 30d max        │  Prometheus: 90d             │  Prometheus: 1 year          │
└─────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
```

---

### Stage A — 1st HDD Added (Single-Disk ZFS)

> ⚠️ ZFS gives you data integrity checksums but NO redundancy yet.
> B2 offsite backup remains mandatory through all of Stage A.

#### Hardware to Buy (Stage A)
```
[ ] 32GB DDR4-2666 UDIMM kit     — MANDATORY before TrueNAS VM
[ ] 1x WD Red Plus 4TB CMR       — note the EXACT SKU (e.g., WD40EFPX)
    Ask seller: "Apakah ini CMR atau SMR?"
    If they don't know, don't buy.
[ ] PCIe SATA card (x1)          — Rp 100,000–200,000
[ ] External 2-bay HDD enclosure — SATA data+power (not USB), Rp 300,000–600,000
    Buy 2-bay now even though only 1 drive — Bay 2 reserved for Stage B
```

#### Step 1: Hardware Installation
```
[ ] Power off HP EliteDesk
[ ] Remove existing RAM, install 32GB kit (2x 16GB DDR4-2666 UDIMM)
[ ] Install PCIe SATA card in PCIe x1 slot
[ ] Connect 2-bay enclosure via SATA data + power cables from SATA card
[ ] Install 1st WD Red Plus 4TB in Bay 1 — leave Bay 2 empty
[ ] Power on → verify Proxmox boots normally
[ ] Proxmox: Node → Disks — new 4TB drive should appear (e.g., /dev/sdb)
```

#### Step 2: Enable VT-d in BIOS
```
[ ] Reboot → F10 → Advanced → Device Options
[ ] Enable VT-d (Intel Virtualization Technology for Directed I/O)
[ ] Save and exit
    Needed for: PCIe passthrough of SATA card → TrueNAS VM
```

#### Step 3: Create TrueNAS Scale VM in Proxmox
```
[ ] Download TrueNAS Scale ISO → upload to Proxmox local storage
[ ] Proxmox UI → Create VM:
    Name:    truenas
    CPU:     2 cores
    RAM:     8192 MB
    Disk:    32 GB on NVMe (TrueNAS OS boot disk only)
    Network: vmbr0 (same bridge as LXCs)
[ ] VM → Hardware → Add → PCI Device
    Select the SATA controller card (NOT the onboard SATA)
    Enable: All Functions, ROM-Bar, Primary GPU: No
[ ] Boot TrueNAS VM → complete installer → set admin password
[ ] Set static IP: 192.168.1.11 (or next available)
[ ] Access TrueNAS UI: http://192.168.1.11
```

#### Step 4: Create Single-Disk ZFS Pool
```
[ ] TrueNAS UI → Storage → Create Pool
    Name:   tank
    Layout: Stripe (single disk — intentional, mirror added in Stage B)
    Disk:   select the 4TB WD Red Plus
    ⚠️ Stripe = no redundancy. This is expected at Stage A.

[ ] Create datasets inside tank:
    tank/immich
    tank/vaultwarden
    tank/sonarqube
    tank/jenkins
    tank/prometheus
    tank/backups
```

#### Step 5: NFS Shares for LXC Access
```
[ ] TrueNAS UI → Sharing → NFS → Add (one share per dataset):
    /mnt/tank/immich      → Networks: 192.168.1.0/24
    /mnt/tank/vaultwarden → Networks: 192.168.1.0/24
    /mnt/tank/sonarqube   → Networks: 192.168.1.0/24
    /mnt/tank/jenkins     → Networks: 192.168.1.0/24
    /mnt/tank/prometheus  → Networks: 192.168.1.0/24
    /mnt/tank/backups     → Networks: 192.168.1.0/24
[ ] Services → NFS → Start + Enable on boot
```

#### Step 6: Mount NFS Shares in Each LXC
```
In each LXC, add mounts to /etc/fstab:
  192.168.1.11:/mnt/tank/immich      /opt/immich/library   nfs  defaults,_netdev  0  0
  192.168.1.11:/mnt/tank/vaultwarden /opt/vaultwarden/data nfs  defaults,_netdev  0  0
  (etc. for each service)

Test before migrating data: mount -a && df -h
Verify each mount appears before proceeding.
```

#### Step 7: Migrate Data (Service by Service)
```
⚠️ Verify B2 backup ran today BEFORE touching any service.

[ ] Vaultwarden (Priority 1 — do first)
    docker stop vaultwarden
    rsync -av /opt/vaultwarden/data/ /mnt/tank/vaultwarden/
    Edit docker-compose.yml: volume path → /mnt/tank/vaultwarden
    docker start vaultwarden → verify login works ✅

[ ] Immich
    docker stop immich
    rsync -av /opt/immich/library/ /mnt/tank/immich/
    Edit docker-compose.yml: volume path → /mnt/tank/immich
    docker start immich → verify photos visible ✅
    Now: import full historical photo library (no longer limited)

[ ] SonarQube + PostgreSQL
    docker stop sonarqube postgresql
    rsync -av /opt/sonarqube/ /mnt/tank/sonarqube/
    Update volume paths → docker start postgresql sonarqube ✅

[ ] Jenkins
    docker stop jenkins
    rsync -av /opt/jenkins/home/ /mnt/tank/jenkins/
    Update volume path → docker start jenkins → verify pipelines ✅

[ ] Prometheus
    docker stop prometheus
    rsync -av /opt/prometheus/data/ /mnt/tank/prometheus/
    Update volume path
    Update retention flag: --storage.tsdb.retention.time=90d
    docker start prometheus → verify metrics flowing ✅
```

#### Step 8: Post-Migration Cleanup
```
[ ] Wait 24 hours — verify ALL services stable before cleanup
[ ] Remove old data dirs from NVMe to reclaim space
    (only after confirming NFS data is intact and services working)
[ ] Update Proxmox backup target:
    Datacenter → Storage → Add → NFS
    Server: 192.168.1.11, Export: /mnt/tank/backups
[ ] Run vzdump backup → verify it lands in tank/backups ✅
[ ] Stage A complete ✅
```

---

### Stage B — 2nd HDD Added (Complete the ZFS Mirror)

> This is the step that delivers real data protection.
> ZFS adds the mirror online — no downtime, no data loss.

#### Hardware to Buy (Stage B)
```
[ ] 1x WD Red Plus 4TB CMR — MUST match Stage A drive exactly
    Check the SKU you noted when buying the 1st drive
    Example: if 1st is WD40EFPX → buy another WD40EFPX
    ⚠️ Different models technically work but increase resilver risk
    ⚠️ Smaller capacity = usable size is capped to the smaller drive
```

#### Step 1: Install 2nd Drive
```
[ ] Power off HP EliteDesk
[ ] Install 2nd WD Red Plus 4TB in Bay 2 of the enclosure
[ ] Power on → verify Proxmox + TrueNAS VM boot normally
[ ] TrueNAS UI → Storage → Disks — new drive should appear
```

#### Step 2: Attach Mirror Online (No Downtime)
```
TrueNAS UI method:
  Storage → tank pool → Manage Devices
  Click the existing VDev → Extend → select new disk → Confirm

CLI method (SSH into TrueNAS VM):
  zpool status tank            ← verify current state (should show stripe)
  zpool attach tank sdb sdc    ← replace with actual disk names
  zpool status tank            ← shows "resilvering" in progress

Resilvering time: 4–8 hours for 4TB
All services stay online during resilvering — no downtime.
Do NOT power off until zpool status shows: mirror  ONLINE
```

#### Step 3: Enable ZFS Snapshots
```
After resilvering completes (zpool status → mirror ONLINE):

[ ] TrueNAS UI → Data Protection → Periodic Snapshot Tasks:
    tank/vaultwarden → every 1 hour,  keep 7 days  ← most critical
    tank/immich      → every 6 hours, keep 7 days
    tank/sonarqube   → daily,         keep 14 days
    tank/jenkins     → daily,         keep 7 days
    tank/prometheus  → daily,         keep 7 days

[ ] Data Protection → Scrub Tasks:
    Add scrub for tank → run monthly
    Scrub detects + auto-repairs silent bit rot
```

#### Step 4: Expand Retention Limits
```
[ ] Prometheus: update --storage.tsdb.retention.time=365d
[ ] Jenkins: update artifact retention → keep last 30 builds
[ ] Grafana: update dashboard retention labels (30d → 1yr)
[ ] Immich: full historical library already imported in Stage A ✅
```

#### Step 5: Final Verification
```
[ ] zpool status tank → confirm:
    NAME    STATE: ONLINE
      mirror  ONLINE
        sdb   ONLINE
        sdc   ONLINE
[ ] Trigger manual snapshot on tank/vaultwarden → verify it appears
[ ] Vaultwarden: login test ✅
[ ] Immich: photo library intact ✅
[ ] B2 backup: keep running
    Mirror protects against drive failure.
    B2 protects against accidental deletion, ransomware, fire, flood.
    Both are complementary — do not remove B2.
[ ] Stage B complete ✅ — full homelab storage protection achieved
```

---

### Emergency: Drive Failure During Resilvering
```
If the new drive shows errors during resilvering:
  zpool detach tank sdc    ← detach the failing drive
  zpool status tank        ← pool returns to stripe (Stage A state)
  All existing data intact ← resilvering was partial, 1st disk untouched
  Return drive to seller → buy same model replacement → repeat Step 2
```

---

### Storage Capacity at Each Stage
```
┌────────────────────────┬──────────────┬──────────────┬──────────────┐
│  Dataset               │  Stage 0     │  Stage A     │  Stage B     │
│                        │  (NVMe only) │  (1x 4TB)    │  (2x 4TB)    │
├────────────────────────┼──────────────┼──────────────┼──────────────┤
│  Immich library        │  ~100 GB     │  ~2 TB       │  ~2 TB       │
│  Vaultwarden           │  2 GB        │  2 GB        │  2 GB        │
│  SonarQube             │  30 GB       │  200 GB      │  200 GB      │
│  Jenkins               │  30 GB       │  200 GB      │  200 GB      │
│  Prometheus retention  │  30 days     │  90 days     │  365 days    │
│  Backups               │  80 GB       │  ~1 TB       │  ~1 TB       │
├────────────────────────┼──────────────┼──────────────┼──────────────┤
│  Total usable          │  ~342 GB     │  ~3.6 TB     │  ~3.6 TB     │
│  Redundancy            │  ❌ None     │  ❌ None     │  ✅ Mirror   │
└────────────────────────┴──────────────┴──────────────┴──────────────┘

Note: ZFS mirror uses 2nd drive entirely for redundancy, not extra space.
      2x 4TB mirror = 4TB usable (same as single disk).
```

---

## 14. Buying Guide (Indonesian E-Commerce)

*Tokopedia / Shopee / Lazada*

### Priority 1 (Buy Now): M.2 NVMe SSD

> Get 500GB — not 256GB. Your full stack needs the headroom.

| Product | Capacity | Est. Price (Rp) | Notes |
|---|---|---|---|
| WD Blue SN570 | 500GB | 450,000–550,000 | ⭐ Recommended — reliable, good value |
| Samsung 980 | 500GB | 550,000–650,000 | Premium option |
| Kingston NV2 | 500GB | 380,000–480,000 | Budget option, acceptable |

**⚠️ Why 500GB and not 256GB:**
```
256GB NVMe allocation:
  Proxmox OS:   32 GB
  All LXC data: ~150 GB
  Backups:      ~50 GB
  ─────────────────────
  Free:         ~24 GB ← dangerously tight

500GB NVMe allocation:
  Proxmox OS:   32 GB
  All LXC data: ~200 GB
  Backups:      ~80 GB
  ─────────────────────
  Free:         ~188 GB ← comfortable headroom
```

**Search:** `SSD NVMe M.2 2280 500GB`, `WD Blue SN570 500GB`

---

### Priority 2 (When HDD Phase Begins): RAM Upgrade

> Not needed now. Required when TrueNAS VM is added.

| Product | Capacity | Est. Price (Rp) | Notes |
|---|---|---|---|
| DDR4-2666 UDIMM 2x16GB kit | 32GB | 400,000–700,000 (used) | Best value |
| DDR4-2666 UDIMM 1x16GB | 16GB (add-on) | 200,000–350,000 (used) | If current is 2x8GB |

**Spec:** DDR4, UDIMM (desktop), non-ECC, 2666MHz (PC4-21300)
**Search:** `RAM DDR4 2666 16GB UDIMM desktop`, `memori DDR4 PC4-2666`

---

### Priority 3 (When HDD Phase Begins): NAS HDD

> You can buy one at a time. **You MUST buy the same model (same SKU) both times.** Note the exact model number (e.g., WD40EFPX) when buying the first drive — you'll need it when buying the second.

| Product | Capacity | Est. Price/drive (Rp) | Notes |
|---|---|---|---|
| **WD Red Plus (CMR)** | 4TB | 800,000–1,000,000 | ⭐ Best choice |
| **Seagate IronWolf** | 4TB | 850,000–1,100,000 | ⭐ Also excellent |

**❌ Avoid:** WD Red non-Plus, WD Purple, Seagate SkyHawk, SMR drives
**Ask:** *"Apakah ini CMR atau SMR?"* — if they don't know, don't buy.

---

### Priority 4 (When HDD Phase Begins): PCIe SATA Card + Enclosure

| Product | Est. Price (Rp) | Notes |
|---|---|---|
| PCIe SATA card 4-port | 100,000–200,000 | PCIe x1 slot on EliteDesk |
| External 2-bay HDD enclosure | 300,000–600,000 | SATA data+power (not USB) |

---

### Priority 5: Domain + VPS

| Item | Cost | Where |
|---|---|---|
| Domain already bought | — | RumahWeb → nameservers → Cloudflare |
| Hetzner VPS CAX11 | ~€3.29/month (~Rp 57,000) | hetzner.com → Singapore |

---

### Budget Summary

```
┌────────────────────────────────────────┬──────────────────────────┐
│  Phase                                 │  Est. Cost (Rp)          │
├────────────────────────────────────────┼──────────────────────────┤
│  NOW: M.2 NVMe 500GB                   │  380,000–650,000         │
│  LATER: 32GB RAM kit (used)            │  400,000–700,000         │
│  LATER: 2x WD Red Plus 4TB            │  1,600,000–2,000,000     │
│  LATER: PCIe SATA card + enclosure     │  400,000–800,000         │
│  ONGOING: Hetzner VPS (Phase 2)        │  ~57,000/month           │
├────────────────────────────────────────┼──────────────────────────┤
│  Start today (NVMe only)               │  Rp 380,000–650,000 ✅   │
│  Full setup (NVMe + HDD + RAM)         │  Rp 2,780,000–4,150,000  │
└────────────────────────────────────────┴──────────────────────────┘
```

---

## Quick Reference Cheat Sheet

```
Network:        IndiHome CGNAT → Huawei HG8145V5 → pfSense → LAN
Server:         HP EliteDesk 800 G4 SFF (i5-8500, 16GB RAM now → 32GB later)
Hypervisor:     Proxmox VE (bare metal)

Storage NOW:    500GB M.2 NVMe only — no redundancy
                Offsite backup via Rclone → Backblaze B2 is MANDATORY

Storage LATER:  2x 4TB WD Red Plus → ZFS mirror → TrueNAS Scale VM
                Add when budget allows — migrate data with steps in Section 13

Services:
  Core:         Immich (new photos only), Vaultwarden, Stirling-PDF, Uptime Kuma
                AdGuard Home (backup DNS)
  Dev/CI:       GitHub (remote) + Jenkins + SonarQube (run on-demand if RAM pressure)
  IaC:          Terraform CLI, Ansible CLI (no RAM cost when idle)
  Monitoring:   Prometheus (30d retention), Grafana
  DNS/Blocking: pfBlockerNG on pfSense (primary) + AdGuard Home LXC (backup)
  Proxy:        Nginx Proxy Manager

Remote Phase 1: Tailscale (private) + Cloudflare Tunnel (public URLs)
Remote Phase 2: WireGuard VPS relay → sovereignty + OpenVPN restored

Cloud Staging:  AWS t3.micro / GCP e2-micro
                Terraform provisions → Ansible configures → Jenkins deploys

Domain:         RumahWeb (bought) → Cloudflare (DNS manager)
VPS:            Hetzner Singapore (~€3.29/month) — Phase 2

⚠️  NVMe-only rules:
    → Prometheus max 30-day retention
    → Jenkins keep last 10 builds only
    → Immich: new photos only until HDD arrives
    → Uptime Kuma disk alert at 70%
    → Vaultwarden backed up to B2 daily — non-negotiable
```

---

*Guide compiled based on your specific setup: IndiHome CGNAT + HP EliteDesk 800 G4 SFF + pfSense LAN*
*HP hardware specs sourced from official HP Maintenance and Service Guide (Document: c06472102)*
*Stack profile: Solo developer, DevOps-grade homelab, CI/CD + cloud staging, family photo/password management*
*Current phase: NVMe-only starter — TrueNAS + ZFS added in HDD migration phase*