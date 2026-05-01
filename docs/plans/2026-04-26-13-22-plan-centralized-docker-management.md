# Plan: Centralized Docker Management Across LXC Containers
**Date:** April 26, 2026
**Status:** Draft / Under Review

---

## Table of Contents
1. [Short Answer](#1-short-answer)
2. [Why This Works — The Architecture](#2-why-this-works--the-architecture)
3. [Tool Comparison](#3-tool-comparison)
4. [Recommendation: Portainer CE + Agents](#4-recommendation-portainer-ce--agents)
5. [Proposed Architecture](#5-proposed-architecture)
6. [Implementation Plan](#6-implementation-plan)
7. [Resource Budget](#7-resource-budget)
8. [Trade-offs & Caveats](#8-trade-offs--caveats)
9. [Rejected Alternatives](#9-rejected-alternatives)
10. [Open Questions](#10-open-questions)

---

## 1. Short Answer

**Yes — 100% possible, and it's the standard pattern for this kind of setup.**

One Portainer Server instance on one LXC can manage every Docker-enabled LXC
container in your homelab via **Portainer Agents**.

**Concrete fact:**
> "A single Portainer Server will accept connections from any number of Portainer
> Agents, providing the ability to manage multiple clusters from one centralized
> interface."
> — [Portainer Architecture Docs](https://docs.portainer.io/start/architecture)

Portainer CE (free, open source, zlib license, 37k GitHub stars) has **no
environment/node count limit**. You do not need to pay for Business Edition for
this use case.

---

## 2. Why This Works — The Architecture

### The Core Mechanism

Portainer uses a **Server + Agent** model:

```
┌──────────────────────────────────┐       LAN (192.168.1.x)
│  Portainer SERVER                │ ◄─────────────────────────────────┐
│  LXC: Core Services (CT101)      │                                   │
│  Port 9443 (UI + API)            │        port 9001 ◄────────────────┤
│  Port 8000 (Edge tunnel)         │                                   │
└──────────────────────────────────┘                                   │
                                                                       │
                         ┌──────────────────────────────────────────────┤
                         │          Portainer AGENTS (lightweight)      │
                         │                                              │
              ┌──────────▼──────────┐          ┌──────────────────────▼──────┐
              │  LXC: CT100         │          │  LXC: CT102 (future)        │
              │  Network Services   │          │  Dev / CI-CD                │
              │  Portainer Agent    │          │  Portainer Agent             │
              │  (manages NPM)      │          │  (manages Jenkins, etc.)    │
              └─────────────────────┘          └────────────────────────────┘
```

Each Portainer Agent:
- Is a **single lightweight Docker container** (~50MB RAM, ~10MB image)
- Communicates back to the Server over **port 9001 (LAN only)**
- Is **stateless** — all data lives in the Server
- Requires Docker to be installed on the LXC where it runs

The Portainer Server **pulls data from agents** — it reaches out to port 9001 on
each registered environment. This is why all containers must be on the same LAN
(which your Proxmox setup already satisfies).

---

## 3. Tool Comparison

| Criteria | **Portainer CE** | **Coolify** | **Dokploy** |
|---|---|---|---|
| **Multi-host/server** | ✅ Agent-based | ✅ SSH-based | ✅ SSH-based |
| **Free (no node limits)** | ✅ Unlimited | ✅ Free | ✅ Free (self-hosted) |
| **License** | zlib (permissive) | AGPL-3.0 | ⚠️ Proprietary (changed 3mo ago) |
| **RAM footprint** | ~100–200 MB server | ~500 MB+ | ~400 MB+ |
| **Primary purpose** | Docker management UI | Full PaaS | Full PaaS |
| **Git push-to-deploy** | ❌ No | ✅ Yes | ✅ Yes |
| **Built-in reverse proxy** | ❌ No | ✅ Caddy | ✅ Traefik |
| **Manages system services** | ❌ Docker-only | ❌ Docker-only | ❌ Docker-only |
| **Learning curve** | Low | Medium | Medium |
| **Works with your NPM** | ✅ Sits alongside it | ⚠️ Conflicts w/ NPM | ⚠️ Conflicts w/ NPM |
| **Maturity / GitHub stars** | 37.3k ⭐ (8+ years) | 38k ⭐ (active) | 33.5k ⭐ (2 years) |
| **Source** | [github](https://github.com/portainer/portainer) | [site](https://coolify.io/docs) | [github](https://github.com/Dokploy/dokploy) |

### Why Coolify and Dokploy Are NOT Recommended Here

Both Coolify and Dokploy are **full PaaS platforms** (Heroku/Vercel replacements).
They bundle their own reverse proxy (Caddy or Traefik). This creates a **direct
conflict** with your existing Nginx Proxy Manager on CT100.

- You'd end up with two competing reverse proxies routing the same traffic — a
  complex and fragile setup.
- Dokploy recently (3 months ago) introduced a proprietary license for new
  features. Source: [LICENSE.MD on GitHub](https://github.com/Dokploy/dokploy/blob/canary/LICENSE.MD)

**Neither replaces your need for NPM.** But both would duplicate it.

Coolify's SSH-based approach is also a different model: it SSH-es into each server
and runs Docker commands directly. This means **Coolify must be given SSH access
to every LXC** — a larger attack surface than Portainer's on-LAN agent-to-server
pull model.

---

## 4. Recommendation: Portainer CE + Agents

**Verdict:** Keep Portainer Server on CT101. Add a Portainer Agent to every
other LXC that runs Docker.

### Why This Fits Your Setup Perfectly

1. **Already in your Phase 2 plan.** Portainer on CT101 is already planned.
   You're extending it, not replacing it.

2. **Zero conflict with NPM.** Portainer has no built-in proxy. It sits
   alongside NPM without touching port 80/443.

3. **Fits your 16GB RAM budget.** Portainer Server: ~150MB. Each Agent: ~50MB.
   Total overhead for 4 LXC containers: ~300MB total. Negligible.

4. **All on LAN.** Port 9001 agent communication never leaves your Proxmox
   network. No internet exposure required.

5. **Pure visibility + control.** You keep your existing compose files. Portainer
   just gives you a UI to manage them. It doesn't try to own your deployment
   workflow (unlike Coolify/Dokploy which want to be the source of truth).

6. **AdGuard is unaffected.** CT100 runs AdGuard as a system service (not
   Docker). The Portainer Agent on CT100 only manages Docker containers on that
   LXC — which is just NPM. AdGuard is invisible to Portainer (and that's fine).

---

## 5. Proposed Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              PROXMOX VE                                    │
│                          (192.168.1.10:8006)                               │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  LXC: Network Services — CT100                                      │  │
│  │  IP: 192.168.1.100                                                  │  │
│  │                                                                     │  │
│  │  [AdGuard Home — system service, port 53]  ← NOT Docker, invisible  │  │
│  │  [Nginx Proxy Manager — Docker]            ← managed by Agent       │  │
│  │  [Portainer Agent — Docker, port 9001]                              │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                    ▲ port 9001                                              │
│  ┌─────────────────┼───────────────────────────────────────────────────┐  │
│  │  LXC: Core Services — CT101                                         │  │
│  │  IP: 192.168.1.101                                                  │  │
│  │                                                                     │  │
│  │  [Portainer SERVER — Docker, port 9443 UI]  ← Single pane of glass  │  │
│  │  [Vaultwarden — Docker]                                             │  │
│  │  [Immich — Docker]                                                  │  │
│  │  [Stirling-PDF — Docker]                                            │  │
│  │  [Uptime Kuma — Docker]                                             │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                    ▲ port 9001 (future)                                     │
│  ┌─────────────────┼───────────────────────────────────────────────────┐  │
│  │  LXC: Dev / CI-CD — CT102 (future)                                  │  │
│  │  IP: 192.168.1.102                                                  │  │
│  │                                                                     │  │
│  │  [Portainer Agent — Docker, port 9001]                              │  │
│  │  [Jenkins — Docker]                                                 │  │
│  │  [SonarQube — Docker]                                               │  │
│  │  [Gitea — Docker]                                                   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                    ▲ port 9001 (future)                                     │
│  ┌─────────────────┼───────────────────────────────────────────────────┐  │
│  │  LXC: Monitoring — CT103 (future)                                   │  │
│  │  IP: 192.168.1.103                                                  │  │
│  │                                                                     │  │
│  │  [Portainer Agent — Docker, port 9001]                              │  │
│  │  [Prometheus — Docker]                                              │  │
│  │  [Grafana — Docker]                                                 │  │
│  │  [Loki — Docker]                                                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘

From your browser:
  https://portainer.yourdomain.local  →  NPM  →  CT101:9443 (Portainer UI)
  One UI to see all 4 LXCs and all Docker containers inside them.
```

### What You See in Portainer After Setup

Portainer's left sidebar shows **Environments**:

```
Environments
  ├── CT101: Core Services   (local — Portainer's own Docker)
  ├── CT100: Network Svc     (agent — NPM visible here)
  ├── CT102: Dev/CI-CD       (agent — Jenkins, SonarQube visible)
  └── CT103: Monitoring      (agent — Prometheus, Grafana visible)
```

Click any environment → full Docker management: containers, images, volumes,
networks, stacks (compose), logs, exec shell, resource stats.

---

## 6. Implementation Plan

### Phase A — NOW (with your current Phase 2 setup)

**Step A1: Portainer Server on CT101 (already in Phase 2 plan — no change)**

```bash
# On CT101 — already planned in phase-2-network-core-services.md
docker volume create portainer_data
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:sts
```
Source: [docs.portainer.io/start/install-ce/server/docker/linux](https://docs.portainer.io/start/install-ce/server/docker/linux)

**Step A2: Portainer Agent on CT100 (add to your Phase 2 steps)**

> CT100 already runs Docker (for NPM). Agent is ~50MB.

```bash
# On CT100 (SSH into CT100 from Proxmox shell)
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:sts
```
Source: [docs.portainer.io/admin/environments/add/docker/agent](https://docs.portainer.io/admin/environments/add/docker/agent)

**Step A3: Register CT100 as an environment in Portainer UI**

1. Open `https://192.168.1.101:9443` (Portainer UI)
2. Go to **Environments** → **Add environment**
3. Choose **Docker Standalone** → **Agent**
4. Enter:
   - Name: `CT100 Network Svc`
   - Environment URL: `192.168.1.100:9001`
5. Click **Connect**

CT100's Docker containers (NPM) now appear in the Portainer UI.

**Step A4: Verify in Portainer**
- You should see 2 environments: `local` (CT101) and `CT100 Network Svc`
- Click CT100 → Containers → you'll see the NPM container

---

### Phase B — FUTURE (when you add Dev/CI-CD and Monitoring LXCs)

Repeat the same pattern for each new Docker-enabled LXC:

```bash
# On each new LXC (CT102, CT103, etc.)
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:sts
```

Then register in Portainer UI → Add environment → Agent → IP:9001.

**No Portainer Server reinstall needed. Zero downtime.**

---

### Phase C — DNS + Reverse Proxy for Portainer UI

Add to Nginx Proxy Manager on CT100:

```
portainer.yourdomain.local  →  192.168.1.101:9443  (HTTPS)
```

And add the DNS override in pfSense:
```
portainer.yourdomain.local  →  192.168.1.100  (NPM's IP)
```

---

## 7. Resource Budget

| Component | RAM | Storage | Where |
|---|---|---|---|
| Portainer Server | ~150 MB | ~200 MB (volume) | CT101 |
| Portainer Agent (CT100) | ~50 MB | ~20 MB | CT100 |
| Portainer Agent (CT102, future) | ~50 MB | ~20 MB | CT102 |
| Portainer Agent (CT103, future) | ~50 MB | ~20 MB | CT103 |
| **Total overhead** | **~300 MB** | **~260 MB** | all LXCs |

CT101 already has 4GB RAM. 150MB is 3.75% — completely negligible.
CT100 has 1GB RAM. 50MB is 5% — fine.

**No need for a dedicated Management LXC.** That would waste ~512MB–1GB RAM
just to host Portainer Server alone. Keeping it on CT101 is the right call for
your 16GB constraint.

---

## 8. Trade-offs & Caveats

### What Portainer CE Cannot Do

| Limitation | Impact on your setup |
|---|---|
| No Git push-to-deploy | ❌ You handle CI/CD separately via Jenkins (already your plan) |
| No RBAC (role-based access) | ❌ Fine for personal homelab — you're the only user |
| No built-in SSL management | ❌ You already have NPM for that |
| No database management UI | ❌ Out of scope for Docker management tool |
| No system service visibility | ❌ AdGuard (system service) is not visible — but it has its own UI |

None of these are blockers for your homelab. They are all either irrelevant or
already covered by other tools in your stack.

### Port 9001 Is Only LAN-Exposed

Port 9001 (agent port) is used only on the internal LAN (192.168.1.x).
It is NOT exposed to the internet. This is secure for your setup — pfSense
blocks inbound internet traffic anyway, and you're behind CGNAT.

Do NOT add a port-forward rule in pfSense for port 9001.

### CT100 Runs Docker — Verify Before Adding Agent

CT100 is described in Phase 2 as "system service based" but it DOES run Docker
(for NPM). Verify Docker is running on CT100:
```bash
docker ps  # run inside CT100
```
If Docker isn't installed yet, install it before adding the agent (see Phase 2
guide, the Docker install section).

---

## 9. Rejected Alternatives

### Option: Coolify
**Why rejected:**
- Bundles its own Caddy reverse proxy — direct conflict with your NPM on CT100
- SSH-based multi-server means Coolify needs SSH credentials to all your LXCs —
  larger attack surface than Portainer's LAN-only agent model
- Heavier (~500MB) — wasteful when you don't need its PaaS features
- You already have Jenkins for CI/CD; duplicating that with Coolify's Git
  integration creates confusion about "who owns deployments"

Coolify is excellent for teams that want a Heroku-like experience. For a
homelab with separate CI/CD, it's over-engineered.

Source: [coolify.io/docs](https://coolify.io/docs) — "Deploy to any server via SSH"

### Option: Dokploy
**Why rejected:**
- Changed license to proprietary 3 months ago — future paid-features lock-in risk
  Source: [LICENSE.MD](https://github.com/Dokploy/dokploy/blob/canary/LICENSE.MD)
- Same proxy conflict problem as Coolify (uses Traefik)
- Younger project (2 years vs Portainer's 8+ years) — less battle-tested
- Same SSH-based multi-server as Coolify

### Option: Dedicated Management LXC (CT102 just for Portainer)
**Why rejected:**
- Wastes 512MB–1GB RAM for an LXC that runs only one 150MB container
- You're already RAM-constrained at 16GB
- CT101 (Core Services) is the right home for Portainer Server — it's a
  core service

### Option: Docker Socket Proxy / TCP
**Why considered, why rejected:**
- Exposing Docker via TCP socket requires careful TLS cert management
- Portainer Agent over LAN is simpler, more secure, and officially supported
- No benefit for a single-node Proxmox setup on a LAN

---

## 10. Open Questions

> These need clarification before finalizing Phase 2 and this plan.

1. **Does CT100 run Docker for NPM, or is NPM installed as a system service?**
   The Phase 2 guide shows NPM as Docker, but CT100 is labeled "system service
   based." Portainer Agent needs Docker to be present on CT100.

2. **How many total LXC containers are you planning eventually?**
   The homelab guide mentions: Network Svc (CT100), Core Svc (CT101),
   Dev/CI-CD (CT102?), Monitoring (CT103?), Tunnel/Access (CT104?).
   Are all of them Docker-based, or are some system-service only?

3. **Do you want to see Portainer's environment view from a single domain
   (e.g., `portainer.yourdomain.local`)?** This requires a DNS entry + NPM
   proxy rule. Should I include this in Phase 2 or a separate phase?

4. **Is there a plan to run multiple users on Portainer (e.g., a friend/teammate
   with limited access)?** If yes, Portainer Business Edition's RBAC becomes
   relevant — but it's free for up to 3 nodes via the "Take3" offer
   ([portainer.io/take-3](https://www.portainer.io/take-3)).

5. **Do you want Portainer to also manage Proxmox-level containers (via the
   Proxmox API), or only Docker containers inside LXCs?** Portainer does not
   integrate with Proxmox's LXC management — that stays in the Proxmox UI at
   port 8006. This plan only covers Docker containers inside each LXC.
