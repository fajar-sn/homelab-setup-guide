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
11. [Monitoring Stack & Notifications](#11-monitoring-stack--notifications)
12. [IoT VLAN & Home Automation](#12-iot-vlan--home-automation)
13. [Full Roadmap](#13-full-roadmap)
14. [HDD Migration Plan](#14-hdd-migration-plan)
15. [Buying Guide (Indonesian E-Commerce)](#15-buying-guide-indonesian-e-commerce)

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
⚠️ Syncthing + Memos add ~100MB to Core LXC — negligible, 4GB budget unchanged.
⚠️ Home Assistant HaOS VM needs 2–4GB → DO NOT add HA until RAM upgraded to 32GB.
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
                           Traefik
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
│  │  │  │ Traefik               │    │  │  → WireGuard (Ph.2)   │    │  │
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
│    └── Syncthing data             │  2 GB               │
│    └── Memos data                 │  1 GB               │
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
│  Free headroom                    │  ~100 GB            │
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
                   Traefik → services
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

Tool:    restic  (NOT rclone — critical difference explained below)
Backend: Backblaze B2 with Object Lock enabled

WHY restic, NOT rclone:
  rclone is a sync tool — it mirrors, it does NOT version:
    Local: vault.db corrupts at 11:58 PM
    rclone cron runs at midnight
    B2 bucket: vault.db (corrupted) — good copy OVERWRITTEN ❌

  restic is a backup tool — every run creates a new snapshot:
    Local: vault.db corrupts at 11:58 PM
    restic cron runs at midnight
    B2 bucket: snapshot-001 (healthy, yesterday) ← STILL EXISTS ✅
               snapshot-002 (corrupted, today)
    Restore: restic restore snapshot-001 → vault.db recovered ✅

WHY B2 Object Lock (ransomware + accidental delete protection):
  Without Object Lock:
    Attacker steals B2 API key → deletes all snapshots → gone forever ❌
  With Object Lock (WORM — Write Once Read Many):
    B2 REFUSES delete — locked objects cannot be removed by anyone,
    including the account owner. Same model used by healthcare/finance
    for tamper-proof retention. ✅

┌──────────────────────────────────────────────────────────────────┐
│  What MUST be backed up offsite (B2 + Object Lock)               │
├──────────────────────────────────────────────────────────────────┤
│  Priority 1 — Vaultwarden /data/db.sqlite3                       │
│               → restic backup daily → B2 (Object Lock)          │
│               → If lost: all passwords gone forever             │
│               → restic policy: keep 30 daily, 12 monthly        │
├──────────────────────────────────────────────────────────────────┤
│  Priority 2 — Immich /library                                    │
│               → restic backup weekly → B2 (Object Lock)         │
│               → Photos are irreplaceable                        │
│               → restic dedup: only changed chunks upload        │
├──────────────────────────────────────────────────────────────────┤
│  Priority 3 — Vaultwarden offline backup (USB)                   │
│               → Monthly export: Vaultwarden Admin               │
│                 Panel → Export → encrypted JSON                 │
│               → Copy to USB drive → store securely offline      │
│               → Protects against: B2 outage, corruption,        │
│                 accidental cloud delete, ransomware             │
├──────────────────────────────────────────────────────────────────┤
│  Priority 4 — Proxmox LXC backup (vzdump)                        │
│               → Weekly to local NVMe /backup dir                │
│               → Allows fast LXC restore without internet        │
└──────────────────────────────────────────────────────────────────┘

Backblaze B2 paid: ~$1.20/month at 200GB (cheaper than Google Drive 200GB)
B2 Object Lock:    enable per-bucket in B2 console — Governance or Compliance mode

restic quickstart:
  restic -r b2:your-bucket init                       ← create repo (one-time)
  restic -r b2:your-bucket backup /opt/vaultwarden/data  ← backup
  restic -r b2:your-bucket snapshots                  ← verify snapshots exist
  restic -r b2:your-bucket check                      ← integrity check
  restic -r b2:your-bucket restore latest --target /tmp/restore  ← test restore

Cron (crontab -e):
  0 2 * * *  restic -r b2:your-bucket backup /opt/vaultwarden/data  ← daily 2AM
  0 3 * * 0  restic -r b2:your-bucket backup /opt/immich/library    ← weekly Sunday
  0 4 * * 0  restic -r b2:your-bucket forget --keep-daily 30 --keep-monthly 12 --prune
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
| CI/CD (legacy) | Jenkins | ✅ Install now — learn enterprise Groovy DSL patterns |
| CI/CD (modern) | GitHub Actions self-hosted runner | ✅ Install now — zero RAM idle, portfolio-ready |
| Code quality | SonarQube | ✅ Install now |
| IaC | Terraform CLI | ✅ Install now |
| Config mgmt | Ansible CLI | ✅ Install now |
| Metrics | Prometheus + exporters | ✅ Install now (30d retention) |
| Dashboards | Grafana | ✅ Install now |
| DNS Primary | pfBlockerNG (pfSense) | ✅ Already active — no action needed |
| DNS Backup | AdGuard Home (LXC) | ✅ Install as fallback |
| Reverse proxy | Traefik | ✅ Install now |
| Remote (Phase 1) | Tailscale | ✅ Install now |
| Public URLs | Cloudflare Tunnel | ✅ Install now |
| Remote (Phase 2) | WireGuard on VPS | ⏳ Month 3–6 |
| SMB/NFS shares | ~~TrueNAS~~ | ⏳ Add when HDD arrives |
| File sync | Syncthing | ✅ Install now — phone/device → server sync, replaces Google Drive |
| Quick notes | Memos | ✅ Install now — Google Keep replacement (~50MB RAM) |
| Knowledge mgmt | Obsidian (client) + Syncthing | ✅ Zero server RAM — sync Obsidian vault via Syncthing |
| Home automation | Home Assistant (HaOS VM) | ⏳ After RAM upgrade to 32GB + IoT VLAN setup |
| Office suite | OnlyOffice | ⏳ After RAM upgrade — run on-demand, not 24/7 |
| File sharing (LAN) | Samba | ⏳ Add when HDD arrives — serve files from ZFS pool |
| IoT network isolation | pfSense VLAN + managed switch | ⏳ Before Home Assistant — mandatory security step |
| System alerts | Grafana → Telegram bot + Email | ✅ Set up in Phase 6 monitoring |

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
| **Traefik** | ✅ Replace NPM | Docker-native auto-discovery, CNCF-listed, GitOps-compatible |
| **AdGuard Home** | ✅ Add as backup | Secondary DNS — fallback if pfBlockerNG fails |
| **pfBlockerNG** | ✅ Already active | Primary DNS blocking on pfSense |
| **Grafana + Prometheus** | ✅ Keep | 30-day retention on NVMe |
| **Uptime Kuma** | ✅ Keep | Monitor disk usage alert at 70% |
| **Teleport** | ❌ Replace | Tailscale SSH handles this at zero cost |
| **Gitea** | ❌ Skip | Using GitHub instead — saves 20GB disk + RAM |
| **Jenkins** | ✅ Add | Legacy CI/CD — learn Groovy DSL + enterprise patterns |
| **GitHub Actions runner** | ✅ Add | Modern CI/CD — YAML-native, GitHub-integrated, zero RAM when idle |
| **Syncthing** | ✅ Add | ~50MB RAM idle. P2P encrypted file sync (phone → server). Replaces Google Drive/iCloud for files. BSD-licensed, no central server |
| **Memos** | ✅ Add | ~50MB RAM. Google Keep replacement. Docker-native, clean UI, active development. No heavy stack |
| **Obsidian** | ✅ Client-only | PKM/markdown notes on your devices. Sync vault folder via Syncthing — zero server RAM cost |
| **Home Assistant** | ⏳ After 32GB RAM | HaOS VM (2–4GB). Home automation standard. IoT VLAN + managed switch required first |
| **OnlyOffice** | ⏳ After 32GB RAM | Run on-demand. 2–4GB RAM. Google Docs/Sheets replacement. Not urgent |
| **Samba** | ⏳ With HDD Stage A | SMB file sharing from ZFS pool. Not useful without significant storage capacity |
| **Nextcloud (full)** | ❌ Skip | Too heavy (~1.5GB RAM) for notes-only use. Use Memos + Obsidian instead |
| **Bitwarden (official)** | ❌ Skip | Vaultwarden IS the Bitwarden server — all official Bitwarden clients connect to it natively |

---

### Traefik Setup & NPM → Traefik Migration

**Why Traefik over Nginx Proxy Manager:**

```
NPM (GUI-first):
  Deploy new container
  → open NPM UI
  → click "Add Proxy Host"
  → type hostname, upstream IP, toggle SSL
  → done
  Config lives in a SQLite DB — not in files, not in Git.
  Every new service = manual click. Zero GitOps compatibility.

Traefik (code-first):
  Deploy new container with labels in docker-compose.yml:
    labels:
      - "traefik.http.routers.myapp.rule=Host(`myapp.lan`)"
      - "traefik.http.services.myapp.loadbalancer.server.port=8080"
  → Traefik auto-discovers container, auto-provisions route
  Config lives in docker-compose.yml alongside the service.
  Git commit = config change. Reviewable, rollback-able. ✅
```

**Traefik docker-compose.yml (Network Services LXC):**
```yaml
# /opt/traefik/docker-compose.yml
services:
  traefik:
    image: traefik:v3
    container_name: traefik
    restart: unless-stopped
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      # Let's Encrypt via Cloudflare DNS challenge (for *.lan internal certs)
      - "--certificatesresolvers.cloudflare.acme.dnschallenge=true"
      - "--certificatesresolvers.cloudflare.acme.dnschallenge.provider=cloudflare"
      - "--certificatesresolvers.cloudflare.acme.email=you@example.com"
      - "--certificatesresolvers.cloudflare.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"        # Traefik dashboard (restrict to LAN only)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
    environment:
      - CF_API_TOKEN=${CLOUDFLARE_API_TOKEN}   # stored in .env file
    networks:
      - proxy

networks:
  proxy:
    external: true
```

**Adding a service to Traefik (example: Vaultwarden):**
```yaml
# in Vaultwarden's docker-compose.yml — add labels block:
services:
  vaultwarden:
    image: vaultwarden/server:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.vault.rule=Host(`vault.lan`)"
      - "traefik.http.routers.vault.entrypoints=websecure"
      - "traefik.http.routers.vault.tls.certresolver=cloudflare"
      - "traefik.http.services.vault.loadbalancer.server.port=80"
    networks:
      - proxy   # must be on the same network as Traefik

networks:
  proxy:
    external: true
```
Apply the same label pattern to: Immich, Stirling-PDF, Uptime Kuma, AdGuard Home, Portainer, Grafana, Jenkins, SonarQube.

**Migration steps from NPM to Traefik:**
```
[ ] Step 1 — Deploy Traefik alongside NPM (both running, no downtime)
    cd /opt/traefik && docker compose up -d
    Access Traefik dashboard: http://[Network LXC IP]:8080

[ ] Step 2 — Migrate services one at a time
    For each service:
      a. Add Traefik labels to its docker-compose.yml
      b. Connect it to the "proxy" Docker network
      c. docker compose up -d --force-recreate [service]
      d. Test: curl -H "Host: service.lan" http://[Traefik IP]
      e. Once confirmed working, remove the NPM proxy host for it

[ ] Step 3 — Migrate cloudflared to point to Traefik
    Update cloudflared tunnel config: upstream → http://[Traefik LXC IP]:80

[ ] Step 4 — Stop and remove NPM
    docker compose down   (in NPM directory)
    NPM is fully replaced. ✅

⚠️  Do NOT remove NPM until all services are confirmed working in Traefik.
    Run both in parallel during migration (NPM on port 81 admin, Traefik on 80/443).
```

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
  vault.lan    → [Traefik LXC IP]   (proxies to Vaultwarden)
  immich.lan   → [Traefik LXC IP]
  pdf.lan      → [Traefik LXC IP]
  status.lan   → [Traefik LXC IP]
  adguard.lan  → [Core Services LXC IP]
  memos.lan    → [Traefik LXC IP]
  syncthing.lan → [Core Services LXC IP]
```

---

### Notes Stack — Memos + Obsidian + Syncthing

**Why NOT Nextcloud for notes?**
```
Nextcloud full stack (just for notes):
  MariaDB      ~400MB RAM
  Redis        ~100MB RAM
  PHP-FPM      ~300MB RAM
  nginx        ~50MB RAM
  Nextcloud    ~400MB RAM
  ──────────────────────
  Total:       ~1.2–1.5GB RAM   (just for sticky notes)

Memos (same result):  ~50MB RAM
Obsidian (client):    0 MB server RAM
Syncthing (vault):    ~50MB RAM (already running for photos)
─────────────────────────────────────
Total:                ~50–100MB RAM
```

**Three-layer notes architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1 — Quick capture (Google Keep replacement)              │
│  Memos :5230 → memos.lan (via Traefik)                         │
│  Use case: quick thoughts, links, shopping lists, reminders     │
│  Mobile: Memos PWA or official app (F-Droid)                   │
├─────────────────────────────────────────────────────────────────┤
│  Layer 2 — Long-form knowledge (Notion/Obsidian replacement)    │
│  Obsidian (client app on phone, laptop, desktop)               │
│  Vault: ~/Documents/ObsidianVault/ (synced via Syncthing)      │
│  Zero server RAM — just a folder synced peer-to-peer           │
│  Plugin ecosystem: Dataview, Excalidraw, Calendar, etc.        │
├─────────────────────────────────────────────────────────────────┤
│  Layer 3 — File sync backbone (Google Drive replacement)        │
│  Syncthing :8384 (admin UI, LAN only)                          │
│  Syncs: DCIM/ (photos for Immich) + ObsidianVault/ + Documents/│
│  E2E encrypted, no central server, works offline               │
└─────────────────────────────────────────────────────────────────┘
```

**Syncthing setup quick-start:**
```bash
# docker-compose.yml for Core Services LXC
syncthing:
  image: lscr.io/linuxserver/syncthing:latest
  container_name: syncthing
  environment:
    - PUID=1000
    - PGID=1000
  volumes:
    - ./syncthing/config:/config
    - /data/syncthing:/data   # mount your sync folder
  ports:
    - 8384:8384   # web UI (restrict to LAN)
    - 22000:22000 # sync protocol
    - 21027:21027/udp # discovery
  restart: unless-stopped
```
Install Syncthing on Android via F-Droid → pair via device ID → sync begins automatically.

---

## 9. CI/CD Pipeline Architecture

### Jenkins vs GitHub Actions — Why You Need Both

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Run BOTH — each serves a different purpose                             │
├──────────────────────────────────────┬──────────────────────────────────┤
│  Jenkins (Path A)                    │  GitHub Actions runner (Path B)  │
│  ─── Legacy CI/CD learning ───       │  ─── Modern industry standard ── │
├──────────────────────────────────────┼──────────────────────────────────┤
│  Groovy DSL (Jenkinsfile)            │  YAML (.github/workflows/*.yml)  │
│  ~4GB RAM running 24/7               │  ~50MB idle, only runs on push   │
│  Plugin ecosystem (650+ plugins)     │  Native GitHub integration       │
│  Used by: banks, telecoms, legacy    │  Used by: Shopify, Vercel, most  │
│  enterprise running Jenkins pre-2018 │  modern companies since 2020     │
│  Stack Overflow 2024: declining      │  Stack Overflow 2024: #1 CI/CD   │
│  Learn to understand legacy systems  │  Use for real projects/portfolio │
│  you'll encounter at enterprise jobs │  Skills transfer to any job      │
└──────────────────────────────────────┴──────────────────────────────────┘

RAM impact on your 16GB setup:
  Jenkins always on:      4GB consumed even with zero builds running
  GHA runner idle:        ~50MB — activates only when GitHub dispatches a job
  → Run SonarQube + Jenkins on-demand when RAM pressure hits
  → GHA runner can stay on 24/7 at negligible cost
```

---

### Path A: Jenkins Pipeline Flow

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

### Path B: GitHub Actions Self-Hosted Runner Flow

```
  Developer pushes code
         │
         ▼
  ┌──────────────┐
  │ GitHub       │
  │ (remote)     │
  └──────┬───────┘
         │ triggers .github/workflows/ci.yml
         ▼
  ┌──────────────────────────────────────────────────────────┐
  │  GitHub Actions (cloud-side orchestration)               │
  │  reads workflow YAML → routes job to self-hosted label   │
  └──────────────────────────┬───────────────────────────────┘
                             │ job dispatched to your server
                             ▼
  ┌──────────────────────────────────────────────────────────┐
  │  GHA Self-Hosted Runner (Dev & CI/CD LXC)                │
  │  (~50MB idle — only active when job dispatched)          │
  │                                                          │
  │  Step 1: Checkout                                        │
  │  Step 2: Build                                           │
  │  Step 3: Test                                            │
  │  Step 4: SonarQube Scan                                  │
  │          SONAR_TOKEN → stored as GitHub repo secret ✅   │
  │          Quality Gate: PASS → continue / FAIL → abort   │
  │  Step 5: Deploy (only on push to main)                   │
  └──────────────────────────┬───────────────────────────────┘
                             │
                    ┌────────┴────────┐
                    ▼                 ▼
                Terraform         Ansible
                apply             deploy.yml
                    └────────┬────────┘
                             ▼
                   staging.yourdomain.com ✅
```

### GitHub Actions Workflow Sample
```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-test-scan:
    runs-on: self-hosted        # routes to your homelab runner
    steps:
      - uses: actions/checkout@v4

      - name: Build
        run: mvn clean package -DskipTests

      - name: Test
        run: mvn test

      - name: SonarQube Scan
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: http://192.168.1.x:9000   # internal LXC IP
        run: mvn sonar:sonar -Dsonar.login=$SONAR_TOKEN

  deploy:
    needs: build-test-scan
    if: github.ref == 'refs/heads/main'
    runs-on: self-hosted
    steps:
      - name: Deploy to Staging
        run: ansible-playbook -i inventory/staging deploy.yml
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

## 11. Monitoring Stack & Notifications

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
```

### Notification Channels — Telegram + Email

**Use BOTH:** different failure modes require different channels.

```
┌─────────────────────────────────────────────────────────┐
│  Telegram Bot — primary, instant, interactive           │
│  ✅ Works in Indonesia (no Google account needed)       │
│  ✅ < 5 min to set up                                   │
│  ✅ Rich alerts: emoji, markdown, service buttons       │
│  ✅ Works even if home server is unreachable            │
│     (sent via Telegram's API from Grafana)              │
├─────────────────────────────────────────────────────────┤
│  Email — backup, delivery guaranteed                    │
│  ✅ Permanent audit log (Gmail search for old alerts)   │
│  ✅ Works when Telegram is blocked or down              │
│  ✅ Required for critical security alerts               │
│  Use: SMTP relay (Gmail + App Password or Mailgun free) │
└─────────────────────────────────────────────────────────┘

Rule: WARNING → Telegram only
      CRITICAL → Telegram + Email
```

**Telegram bot setup (5 minutes):**
```
1. Open Telegram → search @BotFather → /newbot
2. Name it (e.g. HomelabBot) → get TOKEN
3. Start a chat with your bot → get your CHAT_ID:
   curl https://api.telegram.org/bot<TOKEN>/getUpdates
4. In Grafana → Alerting → Contact points → Add Telegram
   Bot token: <TOKEN>
   Chat ID:   <CHAT_ID>
5. Test → you'll get "Test alert" message instantly
```

**Email (Gmail App Password) setup:**
```
Gmail → Settings → Security → 2FA on → App Passwords → Generate
In Grafana → Alerting → Contact points → Add Email
  SMTP host:     smtp.gmail.com:587
  From address:  your@gmail.com
  SMTP user:     your@gmail.com
  SMTP password: [16-char app password]
```

---

## 12. IoT VLAN & Home Automation

### Why IoT VLAN is Mandatory Before Home Assistant

```
⚠️  Smart devices (lights, plugs, cameras) have poor security:
    - hardcoded credentials
    - unpatched firmware
    - phone-home to vendor servers
    - some have known backdoors

  Putting them on your main LAN = they can reach your server at
  192.168.1.10, your NAS, your pfSense admin UI.

  IoT VLAN solution:
    IoT devices get their own subnet (192.168.10.x)
    Firewall rules BLOCK them from reaching 192.168.1.x
    Home Assistant (on trusted VLAN) CAN reach IoT devices
    IoT devices CANNOT reach HA or any server
```

### Hardware Required

| Item | Product | Est. Price (Rp) | Notes |
|---|---|---|---|
| Managed switch | TP-Link TL-SG108E | 350,000–450,000 | 8-port gigabit, 802.1Q VLAN |
| VLAN-capable AP | TP-Link EAP225 | 400,000–600,000 (used) | Multiple SSIDs + VLAN tagging |

> Your current unmanaged switch and Huawei HG8145V5 WiFi cannot do VLAN tagging.
> The Huawei ONT must stay for GPON/IndiHome, but its WiFi can be disabled once you have a proper AP.

### VLAN Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  pfSense (192.168.1.1)                                         │
│  ├─ VLAN 1  (trusted)  192.168.1.x/24   — server, your devices│
│  └─ VLAN 10 (IoT)      192.168.10.x/24  — smart plugs, cameras│
├─────────────────────────────────────────────────────────────────┤
│  pfSense firewall rules for IoT VLAN (192.168.10.x):          │
│  ALLOW   IoT → WAN          (firmware updates, cloud APIs)    │
│  ALLOW   192.168.1.x → IoT  (Home Assistant controls devices) │
│  BLOCK   IoT → 192.168.1.x  (IoT cannot reach server/router) │
│  BLOCK   IoT → IoT          (device isolation, no lateral mvt)│
└─────────────────────────────────────────────────────────────────┘

WiFi SSIDs (on EAP225/EAP610 AP):
  "HomeNet"   → VLAN 1  (trusted) — your phone, laptop
  "HomeIoT"   → VLAN 10 (IoT)     — smart devices only
```

### Home Assistant Setup (after 32GB RAM + IoT VLAN done)

```
Verification before installing HA:
  [ ] RAM = 32GB installed
  [ ] Managed switch in place, VLAN tagging working
  [ ] IoT SSID exists and IoT devices cannot ping 192.168.1.10
  [ ] pfSense VLAN 10 firewall rules verified

Install Home Assistant OS (HaOS) as Proxmox VM:
  CPU:  2 cores
  RAM:  4096 MB
  Disk: Import HaOS QCOW2 from github.com/home-assistant/operating-system
  Net:  vmbr0 (VLAN 1 — trusted) so HA can reach IoT devices

Post-install:
  Access: http://homeassistant.local:8123
  HA discovers devices via mDNS on IoT VLAN
  Enable integrations for your specific smart lights/plugs
  Set up automations: lights on sunset, away mode, etc.
```

> This section is FUTO-aligned: the FUTO guide strongly recommends Home Assistant
> as the home automation standard. The key difference is that FUTO's guide assumes
> unlimited RAM — on your 16GB system, HA must wait for the RAM upgrade.

---

## 13. Full Roadmap

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
[ ] Install Traefik (Docker) — see NPM → Traefik guide in Section 8
[ ] Add local DNS overrides in pfSense:
    Services → DNS Resolver → Host Overrides
    → proxmox.lan  → 192.168.1.10
    → vault.lan    → [Traefik LXC IP]
    → immich.lan   → [Traefik LXC IP]
    → pdf.lan      → [Traefik LXC IP]
    → status.lan   → [Traefik LXC IP]
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
[ ] ⚠️ IMMEDIATELY set up restic → Backblaze B2 (Object Lock) backup for Vaultwarden
    restic -r b2:vault-bucket init
    restic -r b2:vault-bucket backup /opt/vaultwarden/data
    restic -r b2:vault-bucket snapshots   ← verify it worked
[ ] ⚠️ Set up monthly offline backup for Vaultwarden:
    Vaultwarden Admin Panel → Export Vault → Encrypted JSON
    → Copy exported file to USB drive → store securely offline
[ ] Deploy AdGuard Home (backup DNS, port 53 + UI :3000)
[ ] Deploy Immich (new photos only — do NOT import full library yet)
[ ] Deploy Stirling-PDF
[ ] Deploy Uptime Kuma
[ ] Add Uptime Kuma disk usage monitor (alert at 70%)

[ ] Deploy Syncthing (file sync from phone/devices to server)
    Port: 8384 (admin UI — restrict to LAN only in Traefik/firewall)
    Install Syncthing on Android: F-Droid or Play Store
    Add your server as a device → share DCIM/ folder
    → Immich can ingest from /data/syncthing/DCIM/ automatically
    Also sync: ObsidianVault/ folder for note-taking

[ ] Deploy Memos (quick notes — Google Keep replacement)
    Port: 5230
    Add Traefik label: memos.lan
    Add pfSense DNS override: memos.lan → [Traefik LXC IP]

[ ] Install Obsidian on all your devices (phone, laptop, desktop)
    Create vault folder in Syncthing shared directory: ~/Documents/ObsidianVault/
    Open Obsidian → Open folder as vault → point to ObsidianVault/
    Zero server config needed — Syncthing handles all sync

[ ] Configure Traefik Docker labels for:
    vault.lan, immich.lan, pdf.lan, status.lan, adguard.lan, memos.lan
    (see Traefik setup in Section 8 — add labels to each service's docker-compose.yml)

[ ] Add pfSense DNS overrides for new services:
    Services → DNS Resolver → Host Overrides
    memos.lan     → [Traefik LXC IP]
    syncthing.lan → [Core Services LXC IP]:8384
```

### Phase 4 — Remote Access (Day 3–4)
```
[ ] Install Tailscale on pfSense
[ ] Enable subnet routing (192.168.1.0/24)
[ ] Install Tailscale on your devices + wife's devices
[ ] Test remote access: Vaultwarden + Immich via Tailscale
[ ] Deploy cloudflared in Network Services LXC
[ ] Point dev.yourdomain.com → Traefik → dev service
```

### Phase 5 — Dev & CI/CD (Week 2)
```
[ ] Create Dev & CI/CD LXC (6GB RAM)
[ ] Deploy SonarQube + PostgreSQL (Docker Compose)

[ ] --- Path A: Jenkins (legacy CI/CD learning) ---
[ ] Deploy Jenkins
[ ] Configure Jenkins → GitHub integration:
    Jenkins → Manage Jenkins → Plugins → GitHub plugin
    Create GitHub Personal Access Token → add to Jenkins credentials
    Configure webhook on GitHub repo → http://[Jenkins IP]:8080/github-webhook/
[ ] Connect Jenkins → SonarQube (SonarQube Scanner plugin)
[ ] Create first Jenkinsfile pipeline: build → test → scan → quality gate
[ ] Set Jenkins artifact retention: keep last 10 builds only

[ ] --- Path B: GitHub Actions self-hosted runner (modern CI/CD) ---
[ ] In Dev LXC, create dedicated runner user:
    useradd -m github-runner && su - github-runner
[ ] Download + install runner (GitHub: repo → Settings → Actions → Runners → New self-hosted runner)
    mkdir actions-runner && cd actions-runner
    curl -o actions-runner-linux-x64.tar.gz -L [URL from GitHub UI]
    tar xzf actions-runner-linux-x64.tar.gz
[ ] Register runner:
    ./config.sh --url https://github.com/[user]/[repo] --token [TOKEN from GitHub UI]
[ ] Install + start as service (run as root):
    ./svc.sh install github-runner
    ./svc.sh start
[ ] Add SONAR_TOKEN as GitHub repo secret:
    Repo → Settings → Secrets and variables → Actions → New repository secret
[ ] Add .github/workflows/ci.yml to repo (see Section 9 GHA sample)
[ ] Verify: push a commit → Actions tab → runner picks up job ✅

[ ] Install Terraform CLI + Ansible CLI
[ ] Test: Terraform plan, Ansible ping
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
[ ] Verify restic backup is running:
    restic -r b2:vault-bucket snapshots   ← must show at least one snapshot
    restic -r b2:vault-bucket check       ← verify integrity, must show no errors
[ ] Test restore (dry run — non-destructive):
    restic -r b2:vault-bucket restore latest --target /tmp/vault-restore
    ls /tmp/vault-restore   ← must show db.sqlite3 file
```

### Phase 8 — Cloud Staging (Month 2)
```
[ ] Create AWS/GCP free tier account
[ ] Write Terraform code for staging environment
[ ] Store state in Terraform Cloud (free)
[ ] Write Ansible deploy.yml
[ ] Add to Jenkins pipeline: deploy to AWS/GCP after quality gate
[ ] Add to GHA workflow: deploy job triggered on push to main (see Section 9 GHA sample)
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

### Phase 10 — IoT VLAN & Home Automation (lowest priority — do when buying smart devices)
```
💡 Only start this phase when you actually plan to buy smart home devices.
   If you have no IoT devices, skip entirely — there is nothing to isolate.

   Prerequisite order:
     RAM upgrade (32GB) → buy IoT hardware + switch/AP → configure VLAN → install Home Assistant
   All three steps are blocked by the RAM upgrade anyway, so they arrive together.

--- Optional now: Pre-configure pfSense (no hardware required) ---
[ ] Configure VLANs in pfSense (do anytime, costs nothing):
    Interfaces → VLANs → Add
      VLAN 1  (trusted):  192.168.1.x   — your devices, server, pfSense
      VLAN 10 (IoT):      192.168.10.x  — smart lights, plugs, cameras
    Assign VLAN 10 as new interface → enable DHCP for 192.168.10.0/24

[ ] pfSense firewall rules for IoT VLAN interface (192.168.10.x):
    ALLOW   IoT (192.168.10.x) → WAN         (internet access for firmware)
    ALLOW   192.168.1.x → IoT VLAN           (Home Assistant controls devices)
    BLOCK   IoT → 192.168.1.x                (IoT cannot reach server/router)
    BLOCK   IoT → IoT                        (device isolation)

--- Buy when ready for smart home devices (alongside RAM upgrade) ---
[ ] Buy managed switch (TP-Link TL-SG108E ~Rp 400,000)
    Must support 802.1Q VLAN tagging
    Connect: ONT → pfSense WAN port
              pfSense LAN port → managed switch
              Server + EAP225 AP → managed switch

[ ] Buy VLAN-capable WiFi AP (TP-Link EAP225 or EAP610 ~Rp 500,000 used)
    Must support multiple SSIDs each mapped to a different VLAN
    Disable WiFi on Huawei HG8145V5 once AP is working

[ ] Configure EAP225 AP via Omada or standalone mode:
    SSID: "HomeNet" → VLAN 1  (trusted)  — your phone, laptop
    SSID: "HomeIoT" → VLAN 10 (IoT only) — smart devices

[ ] Move all smart devices to HomeIoT SSID

[ ] Verify isolation:
    From IoT device: ping 192.168.1.10 → should FAIL ✅
    From your laptop: ping 192.168.10.x IoT device → should work
    From IoT device: ping 8.8.8.8 → should work (internet access)

--- Then install Home Assistant (RAM must be 32GB) ---
[ ] Download HaOS QCOW2 → create Proxmox VM (see Section 12 for full steps)
[ ] Access http://homeassistant.local:8123
[ ] Install integrations for your smart devices
```

---

## 14. HDD Migration Plan

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

#### Step 2: Drive Health Check & Burn-In

> Run before creating the ZFS pool.
> **Do not skip on used drives** — `badblocks` surfaces latent sector failures that SMART alone misses.

> ⚠️ `badblocks -w` is **destructive** — it erases all data on the drive.
> The drive must be empty. This is intentional: burn-in runs before the ZFS pool exists.

```bash
# Install required tools on the Proxmox host
apt install smartmontools tmux fio
# badblocks is part of e2fsprogs — already present on Proxmox
```

**2a — Re-run Extended SMART Self-Test**

> Re-run now that the drive is seated and has been powered on for several hours.
> Temperature-dependent defects sometimes only appear during actual operation.

```bash
# Confirm drive path
lsblk -d -o NAME,TYPE,SIZE,ROTA,TRAN
smartctl -t long /dev/sdb        # replace with your actual drive path

# Wait 60–90 minutes, then check:
smartctl -l selftest /dev/sdb
# Must show: Completed without error
```

**2b — Full Surface Scan (badblocks)**

> Writes 4 byte patterns to every sector, verifies each pass.
> Takes **8–16 hours** for a 4TB HDD. Plan overnight.

```bash
# Run inside tmux so it survives SSH disconnection
tmux new -s burnin

# -w = write mode (4 passes, destructive)  -s = show progress  -v = verbose
# Replace /dev/sdb with your actual drive path
badblocks -wsv /dev/sdb 2>&1 | tee /root/badblocks-sdb-$(date +%Y%m%d).log

# Detach from tmux (keeps running):  Ctrl-B then D
# Reattach later:                    tmux attach -t burnin
```

Expected final line when complete:
```
Pass completed, 0 bad blocks found.
```

If badblocks reports **any** bad blocks → **do not use this drive**. Return it to the seller.

**2c — Monitor Temperature During Burn-In**

```bash
# Run in a separate terminal while burn-in is in progress
watch -n 60 "smartctl -A /dev/sdb | grep -i temp"
# Safe range: 30–55°C under sustained write load
# If ≥ 58°C → pause burn-in, improve enclosure ventilation, then retry
```

**2d — Throughput Sanity Check (fio)**

```bash
# Sequential read — confirms drive performs within expected range
fio --name=seqread --rw=read --direct=1 --ioengine=libaio \
    --bs=1M --numjobs=1 --size=10G --runtime=60 \
    --group_reporting --filename=/dev/sdb
# Expected 7200rpm SAS/SATA HDD: 150–250 MB/s
# If < 80 MB/s → suspect bad cable, enclosure issue, or drive problem
```

**2e — Post-Burn-In SMART Check**

```bash
smartctl --all /dev/sdb | grep -E "Health Status|grown defect|uncorrected"
# Must still show:
#   SMART Health Status: OK
#   Elements in grown defect list: (same or minimally higher than before)
#   uncorrected errors:             0
```

**Burn-In Gate — all 5 must pass before Step 3:**
```
[ ] Extended SMART self-test:    Completed without error
[ ] badblocks:                   0 bad blocks found
[ ] Temperature:                 stayed ≤ 55°C throughout sustained write
[ ] Throughput:                  ≥ 80 MB/s sequential read
[ ] Post-burn-in SMART:          Health OK, uncorrected errors = 0

If badblocks finds bad blocks → return the drive. Do not build a ZFS pool on it.
```

#### Step 3: Enable VT-d in BIOS
```
[ ] Reboot → F10 → Advanced → Device Options
[ ] Enable VT-d (Intel Virtualization Technology for Directed I/O)
[ ] Save and exit
    Needed for: PCIe passthrough of SATA card → TrueNAS VM
```

#### Step 4: Create TrueNAS Scale VM in Proxmox
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

#### Step 5: Create Single-Disk ZFS Pool
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

#### Step 6: NFS Shares for LXC Access
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

#### Step 7: Mount NFS Shares in Each LXC
```
In each LXC, add mounts to /etc/fstab:
  192.168.1.11:/mnt/tank/immich      /opt/immich/library   nfs  defaults,_netdev  0  0
  192.168.1.11:/mnt/tank/vaultwarden /opt/vaultwarden/data nfs  defaults,_netdev  0  0
  (etc. for each service)

Test before migrating data: mount -a && df -h
Verify each mount appears before proceeding.
```

#### Step 8: Migrate Data (Service by Service)
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

#### Step 9: Post-Migration Cleanup
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

#### Step 10: Set Up Samba File Sharing (from ZFS pool)
```
[ ] TrueNAS UI → Sharing → SMB → Add share:
    Path: /mnt/tank/archive
    Name: archive
[ ] TrueNAS → Services → SMB → Start + Enable on boot
[ ] Windows access:  \\192.168.1.11\archive
[ ] macOS access:    smb://192.168.1.11/archive
[ ] Remote access:   via Tailscale VPN only — NEVER expose SMB (port 445) to internet
[ ] Add Syncthing on TrueNAS:
    TrueNAS → Apps → Syncthing (or run in a jail)
    Point to /mnt/tank/media for large file sync from devices
```

#### Step 11: Add Home Assistant VM (once RAM = 32GB + IoT VLAN done)
```
⚠️  Prerequisites — ALL must be true before continuing:
    [ ] RAM upgraded to 32GB ✅
    [ ] Managed switch (802.1Q VLAN) in place ✅
    [ ] IoT VLAN (192.168.10.x) configured in pfSense ✅
    [ ] IoT devices cannot ping 192.168.1.10 ✅
    [ ] EAP225/EAP610 AP with HomeIoT SSID mapped to VLAN 10 ✅

[ ] Download HaOS QCOW2 image:
    https://github.com/home-assistant/operating-system/releases
    Choose: haos_ova-*.qcow2.xz → extract

[ ] Proxmox → Create VM:
    Name: homeassistant
    CPU:  2 cores
    RAM:  4096 MB
    Disk: Import QCOW2 via:
          qm importdisk <vmid> haos_ova-*.qcow2 local-zfs
    Net:  vmbr0 (trusted VLAN 1 — so HA can control IoT devices)

[ ] Boot VM → access http://homeassistant.local:8123 or http://[HA IP]:8123
[ ] Complete onboarding wizard
[ ] HA discovers devices on IoT VLAN via mDNS (pfSense firewall allows 192.168.1.x → IoT)
[ ] Install integrations: Xiaomi, TP-Link Kasa, Tuya, etc. (whatever smart devices you buy)
[ ] Enable Tailscale add-on in HA for remote access to automations
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

```bash
# Quick SMART health check on the new drive from the Proxmox host before mirroring
smartctl --all /dev/sdc        # replace with actual new drive path
# Verify: SMART Health Status: OK, Elements in grown defect list: ≤ 50
# For used drives with high power-on hours, run the extended test first:
#   smartctl -t long /dev/sdc
#   smartctl -l selftest /dev/sdc   (check after ~90 min)
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

## 15. Buying Guide (Indonesian E-Commerce)

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

### Priority 6: IoT VLAN Hardware (lowest priority — buy alongside RAM upgrade, only when getting IoT devices)

| Product | Est. Price (Rp) | Notes |
|---|---|---|
| TP-Link TL-SG108E (managed switch) | 350,000–450,000 | 8-port gigabit, 802.1Q VLAN tagging |
| TP-Link EAP225 or EAP610 (WiFi AP) | 400,000–600,000 (used) | Multi-SSID + VLAN tagging per SSID |

> **Only buy these when you plan to get smart home devices.** Zero IoT devices = zero risk on your current setup.
> pfSense VLAN rules can be pre-configured for free at any time — the hardware only matters when IoT devices physically arrive.

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
│  LATER: IoT VLAN hardware (switch+AP)  │  750,000–1,050,000       │
│  ONGOING: Hetzner VPS (Phase 2)        │  ~57,000/month           │
├────────────────────────────────────────┼──────────────────────────┤
│  Start today (NVMe only)               │  Rp 380,000–650,000 ✅   │
│  Full setup (NVMe + HDD + RAM + IoT)   │  Rp 3,530,000–5,200,000  │
└────────────────────────────────────────┴──────────────────────────┘
```

---

## Quick Reference Cheat Sheet

```
Network:        IndiHome CGNAT → Huawei HG8145V5 → pfSense → LAN
Server:         HP EliteDesk 800 G4 SFF (i5-8500, 16GB RAM now → 32GB later)
Hypervisor:     Proxmox VE (bare metal)

Storage NOW:    500GB M.2 NVMe only — no redundancy
                Offsite backup via restic → Backblaze B2 (Object Lock) is MANDATORY

Storage LATER:  2x 4TB WD Red Plus → ZFS mirror → TrueNAS Scale VM
                Add when budget allows — migrate data with steps in Section 14

Services:
  Core:         Immich (new photos only), Vaultwarden, Stirling-PDF, Uptime Kuma
                AdGuard Home (backup DNS)
                Syncthing (file sync from phone/devices)
                Memos (quick notes — Google Keep replacement, :5230)
  Notes PKM:    Obsidian client + Syncthing vault sync (zero server RAM)
  Dev/CI:       GitHub (remote) + Jenkins (legacy) + GHA runner (modern) + SonarQube (on-demand)
  IaC:          Terraform CLI, Ansible CLI (no RAM cost when idle)
  Monitoring:   Prometheus (30d retention), Grafana
                Alerts: Telegram bot (primary) + Email/Gmail (backup)
  DNS/Blocking: pfBlockerNG on pfSense (primary) + AdGuard Home LXC (backup)
  Proxy:        Traefik (Docker-native auto-discovery, CNCF-listed)

Deferred (need 32GB RAM + IoT VLAN setup first):
  Home Automation: Home Assistant HaOS VM (4GB RAM) — add Phase 3.5 IoT VLAN first
  File sharing:    Samba (serve ZFS pool) — add with HDD Stage A
  Office suite:    OnlyOffice (on-demand only) — not urgent

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