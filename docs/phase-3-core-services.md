# Homelab Setup Guide — Phase 3
## Core Services

**Target:** Deploy all personal services in CT101 via Docker — Portainer, Vaultwarden, Immich, Stirling-PDF, Uptime Kuma
**Timeframe:** Day 2–3 (3–5 hours)
**Prerequisite:** [Phase 2 — Network Services](phase-2-network-services.md) complete (CT101 running, Nginx PM ready at 192.168.1.100:81)
**Outcome:** All personal services accessible via `*.yourdomain.com` internal URLs through Nginx PM
**Next:** [Phase 4 — Remote Access via Tailscale](phase-4-remote-access.md)

---

## Table of Contents

1. [Phase 3 Overview](#phase-3-overview)
2. [Install Docker in CT101](#step-31-install-docker-in-core-services-container)
3. [Install Portainer](#step-32-install-portainer-docker-management-ui)
4. [Set Up Vaultwarden](#step-33-set-up-vaultwarden-password-manager)
5. [Set Up Backblaze B2 Backup](#step-34-set-up-backblaze-b2-backup-for-vaultwarden)
6. [Set Up Immich](#step-35-set-up-immich-photo-library)
7. [Set Up Stirling-PDF](#step-36-set-up-stirling-pdf)
8. [Set Up Uptime Kuma](#step-37-set-up-uptime-kuma-monitoring)
9. [Add Services to Nginx Proxy Manager](#step-38-add-services-to-nginx-proxy-manager)
10. [Test DNS & Access Services](#step-39-test-dns--access-services)
11. [Backup Strategy](#step-310-backup-strategy)
12. [Testing & Verification Checklist](#testing--verification-checklist)
13. [Troubleshooting](#troubleshooting)

---

## Phase 3 Overview

### Services Being Deployed

| Service | Type | Port | Container | Purpose |
|---|---|---|---|---|
| **Portainer CE** | Docker | 9000, 9443 | CT101 | Docker container management UI |
| **Vaultwarden** | Docker | 8080 (internal) | CT101 | Self-hosted password manager |
| **Immich** | Docker | 2283 (internal) | CT101 | Photo library (Google Photos alternative) |
| **Stirling-PDF** | Docker | 8081 (internal) | CT101 | PDF editing tools |
| **Uptime Kuma** | Docker | 3001 (internal) | CT101 | Service uptime monitoring |

### URL Access Map (After Phase 3)

| URL | Service | Underlying Port |
|---|---|---|
| `http://vault.yourdomain.com` | Vaultwarden | CT101:8080 → Nginx |
| `http://immich.yourdomain.com` | Immich | CT101:2283 → Nginx |
| `http://pdf.yourdomain.com` | Stirling-PDF | CT101:8081 → Nginx |
| `http://status.yourdomain.com` | Uptime Kuma | CT101:3001 → Nginx |
| `http://portainer.yourdomain.com` | Portainer | CT101:9000 → Nginx |

> **All `*.yourdomain.com` URLs route through Nginx PM at 192.168.1.100 — not directly to CT101. pfSense resolves them internally via split-DNS.**

### Storage Warning (NVMe-only Setup)

```
Total NVMe: ~500 GB
  Used (Proxmox OS + CT overhead): ~100 GB
  Available for Docker volumes: ~300 GB
  
Immich allocation: ~100 GB
  → DO NOT import your full photo library yet
  → Use only for NEW photos until HDD arrives (Phase HDD Migration)
```

---

### Step 3.1: Install Docker in Core Services Container

SSH into container 101:

```bash
ssh root@192.168.1.101
```

Install Docker:

```bash
apt update && apt install -y docker.io docker-compose
systemctl enable docker
systemctl start docker
```

Verify:

```bash
docker --version
```

**Expected:** `Docker version 24.x.x` or similar ✅

---

### Step 3.2: Install Portainer (Docker Management UI)

> **Goal:** Web UI to manage Docker containers across all services. Access at port 9000.

SSH into container 101:

```bash
ssh root@192.168.1.101
```

Create Portainer directory and compose file:

```bash
mkdir -p /opt/portainer
cd /opt/portainer

cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/data
    networks:
      - portainer-network

networks:
  portainer-network:
    driver: bridge
EOF
```

Start Portainer:

```bash
docker-compose up -d
```

Access from laptop:
```
http://192.168.1.101:9000
```

**First-time setup:**
1. Set admin username + password (min 12 chars)
2. Choose **Get Started** (local Docker environment)
3. You'll see all running containers on CT101 ✅

> **Add to Nginx PM later:** After Step 3.8, add proxy host `portainer.yourdomain.com` → `192.168.1.101:9000`

---

### Step 3.3: Set Up Vaultwarden (Password Manager)

> **Goal:** Self-hosted password manager. Critical service — MUST be backed up to Backblaze B2.

#### 3.3.1: Create Docker Compose for Vaultwarden

```bash
mkdir -p /opt/vaultwarden
cd /opt/vaultwarden
```

Create `docker-compose.yml`:

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    ports:
      - "8080:80"
    environment:
      DOMAIN: https://vault.yourdomain.com
      SIGNUPS_ALLOWED: "false"
      INVITATIONS_ORG_ALLOWED: "false"
      SHOW_PASSWORD_HINT: "false"
      LOG_LEVEL: info
      EXTENDED_LOGGING: "true"
    volumes:
      - ./vw-data:/data
    networks:
      - vw-network

networks:
  vw-network:
    driver: bridge
EOF
```

#### 3.3.2: Start Vaultwarden

```bash
docker-compose up -d
```

Wait ~10 seconds for startup.

#### 3.3.3: Verify Vaultwarden Running

```bash
docker ps | grep vaultwarden
```

**Access from laptop:**
```
http://192.168.1.101:8080
```

You'll see Vaultwarden login page ✅

#### 3.3.4: Create Admin Account

1. In Vaultwarden web UI, click **Create account**
2. Email: your-email@gmail.com
3. Master password: Strong password (min 12 chars)
   - Example: `VaultMaster@2026!Secure`
4. Confirm password
5. Click **Create account**

**Write down:**
- Email: your-email@gmail.com
- Master password: (your chosen password)

#### 3.3.5: Offline USB Backup (Emergency Recovery)

> **Why:** If Backblaze B2 is unavailable or rclone fails, you need a physical backup accessible offline.

1. Log into Vaultwarden → click account name → **Account Settings**
2. Go to **Security** → **Export Vault**
3. Choose **Encrypted JSON** format
4. Enter master password → **Confirm Format** → **Export Vault**
5. Save the downloaded `.json` file to a **USB drive** (keep offline — not normally connected)
6. Label the USB drive clearly
7. Repeat monthly or after any major change

> **Store the USB drive securely.** The encrypted JSON is protected by your master password.

---

### Step 3.4: Set Up Backblaze B2 Backup for Vaultwarden

> **CRITICAL:** Vaultwarden holds all your passwords. If the container dies with no backup, you lose everything.
> **Strategy:** Automatic daily backup to Backblaze B2 (free 10 GB tier).

#### 3.4.1: Create Backblaze B2 Account

1. Go to **backblaze.com** → **Sign up** → Free tier (10 GB free)
2. Verify email

#### 3.4.2: Create B2 Bucket for Backups

1. B2 dashboard → **Buckets** → **Create a Bucket**
2. Bucket name: `homelab-backups`
3. **Type:** Private
4. Create bucket

#### 3.4.3: Generate B2 Application Key

1. B2 dashboard → **Account** → **Application Keys**
2. Click **Create New Application Key**
3. **Capabilities:** Select `listBuckets`, `listFiles`, `readFiles`, `writeFiles`
4. **Bucket restriction:** Select `homelab-backups`
5. Generate key

**You'll see:**
```
Application Key ID:     [copy this]
Application Key:        [copy this]
Bucket ID:              [copy this]
```

**Save these three values in Vaultwarden** (once set up) or in a secure note now.

#### 3.4.4: Install Rclone in Core Services Container

SSH into container 101:

```bash
curl https://rclone.org/install.sh | sudo bash
rclone --version
```

#### 3.4.5: Configure Rclone for B2

```bash
rclone config
```

**Follow prompts:**
- Type `n` (new remote)
- `name>` → type `backblaze-b2`
- `Type of storage>` → type `b2`
- `account_id>` → paste your Application Key ID
- `application_key>` → paste your Application Key
- Confirm: `y`
- Quit: `q`

#### 3.4.6: Verify Rclone Connection

```bash
rclone lsd backblaze-b2:
```

**Expected output:**
```
          -1 2026-04-25 10:00:00     -1 homelab-backups
```

#### 3.4.7: Create Backup Script

```bash
cat > /opt/vaultwarden/backup-to-b2.sh << 'EOF'
#!/bin/bash

# Vaultwarden Backup to Backblaze B2
# Run daily via cron

VAULTWARDEN_DATA="/opt/vaultwarden/vw-data"
BACKUP_DIR="/tmp/vw-backup-$(date +%Y%m%d-%H%M%S)"
B2_PATH="backblaze-b2:homelab-backups/vaultwarden/$(date +%Y-%m-%d)"

# 1. Create backup directory
mkdir -p "$BACKUP_DIR"

# 2. Backup database
docker exec vaultwarden sh -c "tar -czf - /data" > "$BACKUP_DIR/vaultwarden-data.tar.gz"

# 3. Upload to B2
rclone sync "$BACKUP_DIR" "$B2_PATH" --progress --log-level INFO

# 4. Keep last 30 days (delete older backups for THIS path only)
rclone delete backblaze-b2:homelab-backups/vaultwarden --min-age 30d

# 5. Cleanup local backup
rm -rf "$BACKUP_DIR"

echo "Backup to B2 completed: $(date)" >> /var/log/vw-backup.log
EOF

chmod +x /opt/vaultwarden/backup-to-b2.sh
```

#### 3.4.8: Schedule Daily Backup via Cron

```bash
crontab -e
```

Add at the end:
```
0 2 * * * /opt/vaultwarden/backup-to-b2.sh
```

(Runs daily at 2 AM)

Save: Ctrl+O → Enter → Ctrl+X

#### 3.4.9: Test Backup Manually

```bash
/opt/vaultwarden/backup-to-b2.sh
```

**Verify in B2 dashboard:** Buckets → homelab-backups → should see folder `vaultwarden/2026-xx-xx/`

✅ Automated backup configured.

---

### Step 3.5: Set Up Immich (Photo Library)

> **Goal:** Self-hosted photo management (like Google Photos).
> **Version note:** Immich v1.91+ — Typesense was removed; vector search now uses `pgvecto-rs` inside PostgreSQL.
> **Limitation (NVMe-only):** ~100 GB storage max = 2–3 years of photos.
> **Strategy:** Use for NEW photos only. Import full library when HDD arrives.

#### 3.5.1: Create Docker Compose for Immich

```bash
mkdir -p /opt/immich
cd /opt/immich
```

Create `docker-compose.yml`:

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  immich-server:
    image: ghcr.io/immich-app/immich-server:latest
    container_name: immich-server
    restart: always
    ports:
      - "2283:3001"
    environment:
      DB_HOSTNAME: immich-db
      DB_USERNAME: immich
      DB_PASSWORD: immich_secure_password_change_me
      DB_NAME: immich
      REDIS_HOSTNAME: immich-redis
      LOG_LEVEL: log
    volumes:
      - ./library:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    depends_on:
      - immich-db
      - immich-redis
    networks:
      - immich-network

  immich-db:
    image: tensorchord/pgvecto-rs:pg16-v0.2.0
    container_name: immich-db
    restart: always
    environment:
      POSTGRES_USER: immich
      POSTGRES_PASSWORD: immich_secure_password_change_me
      POSTGRES_DB: immich
    volumes:
      - ./db:/var/lib/postgresql/data
    networks:
      - immich-network

  immich-redis:
    image: redis:7-alpine
    container_name: immich-redis
    restart: always
    networks:
      - immich-network

networks:
  immich-network:
    driver: bridge
EOF
```

**Change passwords:** Set `DB_PASSWORD` to the same strong password in both `immich-server` and `immich-db`.

#### 3.5.2: Start Immich

```bash
docker-compose up -d
```

Wait ~30 seconds for all 3 containers to start.

#### 3.5.3: Verify Immich Running

```bash
docker ps | grep immich
```

**Should show 3 containers: server, db, redis**

#### 3.5.4: Access Immich Web UI

From laptop:
```
http://192.168.1.101:2283
```

**First time:**
1. Click **Create account**
2. Email: your-email@gmail.com
3. Password: (strong password)
4. Click **Sign up**

You're now in Immich ✅

#### 3.5.5: ⚠️ Storage Limitation Warning

**On NVMe-only setup:**

| Upload rate | Duration on 100 GB |
|---|---|
| 1 GB/month | ~8 years |
| 5 GB/month | ~20 months |
| 10 GB/month | ~10 months |

**Action:** DO NOT import your full photo library yet. Use Immich only for NEW photos until HDD arrives.

---

### Step 3.6: Set Up Stirling-PDF

> **Goal:** PDF manipulation tool (merge, split, rotate, extract, etc.).
> **Lightweight:** ~200 MB Docker image, minimal resource usage.

#### 3.6.1: Create Docker Compose for Stirling-PDF

```bash
mkdir -p /opt/stirling-pdf
cd /opt/stirling-pdf
```

Create `docker-compose.yml`:

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  stirling-pdf:
    image: frooodle/s-pdf:latest
    container_name: stirling-pdf
    restart: always
    ports:
      - "8081:8080"
    environment:
      DOCKER_ENABLE_SECURITY: "true"
      ALLOWED_HOSTS: "192.168.1.101,pdf.yourdomain.com"
    volumes:
      - ./uploads:/home/stirlingpdf/upload
    networks:
      - pdf-network

networks:
  pdf-network:
    driver: bridge
EOF
```

#### 3.6.2: Start Stirling-PDF

```bash
docker-compose up -d
```

#### 3.6.3: Access Stirling-PDF Web UI

From laptop:
```
http://192.168.1.101:8081
```

You see the Stirling-PDF dashboard with tools: Merge PDFs, Split PDF, Rotate pages, Extract images, etc. ✅

---

### Step 3.7: Set Up Uptime Kuma (Monitoring)

> **Goal:** Monitor service uptime + get alerts when anything goes down.
> **Critical for NVMe-only:** Alert when disk reaches 70% capacity.

#### 3.7.1: Create Docker Compose for Uptime Kuma

```bash
mkdir -p /opt/uptime-kuma
cd /opt/uptime-kuma
```

Create `docker-compose.yml`:

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: always
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
    networks:
      - kuma-network

networks:
  kuma-network:
    driver: bridge
EOF
```

#### 3.7.2: Start Uptime Kuma

```bash
docker-compose up -d
```

#### 3.7.3: Access Uptime Kuma Web UI

From laptop:
```
http://192.168.1.101:3001
```

**First time:**
1. Create admin account with email + password
2. Click **Create account**

#### 3.7.4: Add Monitors

In Uptime Kuma dashboard → **Add New Monitor**:

| Friendly Name | Monitor Type | URL |
|---|---|---|
| Vaultwarden Health | HTTP(s) | `http://192.168.1.101:8080/alive` |
| Immich | HTTP(s) | `http://192.168.1.101:2283/api/server/ping` |
| Stirling-PDF | HTTP(s) | `http://192.168.1.101:8081/health` |
| Nginx PM | HTTP(s) | `http://192.168.1.100:81` |
| AdGuard Home | HTTP(s) | `http://192.168.1.100:3000` |
| Proxmox | HTTP(s) | `https://192.168.1.10:8006` |

For each: **Add New Monitor** → fill Name + URL → Interval: 60 seconds → **Save**

**After adding all monitors, they should show "UP" ✅**

#### 3.7.5: Set Up Disk Usage Alert (NVMe-only)

Create a disk check script:

```bash
cat > /opt/uptime-kuma/check-disk.sh << 'EOF'
#!/bin/bash

# Check NVMe disk usage on Proxmox host
DISK_USAGE=$(ssh root@192.168.1.10 'df /dev/nvme0n1p3 | tail -1 | awk "{print \$5}"' | sed 's/%//')

if [ "$DISK_USAGE" -gt 70 ]; then
  echo "CRITICAL: NVMe usage at ${DISK_USAGE}%"
  exit 1
else
  echo "OK: NVMe usage at ${DISK_USAGE}%"
  exit 0
fi
EOF
chmod +x /opt/uptime-kuma/check-disk.sh
```

**For now:** Monitor disk visually via Proxmox dashboard regularly.

---

## DNS & Reverse Proxy Configuration

### Step 3.8: Add Services to Nginx Proxy Manager

> **Goal:** Route clean `*.yourdomain.com` internal URLs to services via Nginx PM.

#### 3.8.1: Access Nginx PM Admin

Browser:
```
http://192.168.1.100:81
```

Login with credentials set in Phase 2 Step 2.8.5.

#### 3.8.2: Add Proxy Host for Vaultwarden

1. **Proxy Hosts** → **Add Proxy Host**

```
Domain Names:          vault.yourdomain.com
Scheme:                http
Forward Hostname/IP:   192.168.1.101
Forward Port:          8080
Cache Assets:          OFF
Block Common Exploits: ON
```

Click **Save**

#### 3.8.3: Add Proxy Hosts for All Other Services

Repeat **Add Proxy Host** for each:

| Domain | Scheme | Forward IP | Forward Port |
|---|---|---|---|
| `immich.yourdomain.com` | http | `192.168.1.101` | `2283` |
| `pdf.yourdomain.com` | http | `192.168.1.101` | `8081` |
| `status.yourdomain.com` | http | `192.168.1.101` | `3001` |
| `portainer.yourdomain.com` | http | `192.168.1.101` | `9000` |
| `nginx.yourdomain.com` | http | `192.168.1.100` | `81` |
| `adguard.yourdomain.com` | http | `192.168.1.100` | `3000` |

After adding all → **Proxy Hosts** tab shows your full list ✅

---

### Step 3.9: Test DNS & Access Services

#### 3.9.1: Verify DNS Resolution

From your laptop:

```bash
nslookup vault.yourdomain.com
# Expected: Address 192.168.1.100 (internal IP — pfSense split-DNS intercepted it)

nslookup immich.yourdomain.com
# Expected: Address 192.168.1.100
```

If these fail: Check pfSense host overrides are saved and applied with domain `yourdomain.com` (Phase 2 Step 2.6.10).

#### 3.9.2: Access All Services via .lan URLs

Test each URL from your laptop browser:

| URL | Expected |
|---|---|
| `http://vault.yourdomain.com` | Vaultwarden login page |
| `http://immich.yourdomain.com` | Immich web UI |
| `http://pdf.yourdomain.com` | Stirling-PDF dashboard |
| `http://status.yourdomain.com` | Uptime Kuma dashboard |
| `http://portainer.yourdomain.com` | Portainer login |
| `http://nginx.yourdomain.com` | Nginx PM admin |
| `http://adguard.yourdomain.com` | AdGuard Home admin |

All should load through Nginx PM ✅

---

### Step 3.10: Backup Strategy

> **Summary:** What gets backed up, where, and how often.

#### Automated Backups (Backblaze B2)

| Service | Data | Frequency | Method |
|---|---|---|---|
| Vaultwarden | `vw-data/` | Daily at 2 AM | rclone → B2 |
| Immich | `library/` | Weekly (planned) | rclone → B2 |

**Backblaze B2 path structure:**
```
homelab-backups/
  ├── vaultwarden/
  │   ├── 2026-04-25/
  │   ├── 2026-04-26/
  │   └── ... (last 30 days)
  └── immich/
      └── ... (last 4 weeks, when configured)
```

**B2 free tier:** 10 GB — Vaultwarden data is small (~100 MB). Immich uploads use more space.

#### Manual Backups

| Service | Method | Frequency |
|---|---|---|
| Vaultwarden vault | Export encrypted JSON → USB | Monthly |
| All containers | Proxmox VZDump snapshot | Before major changes |

#### Proxmox-level Backups

```bash
# Manual snapshot of a container
vzdump 100 --compress gzip --storage proxmox-local-backup
vzdump 101 --compress gzip --storage proxmox-local-backup
```

---

## Testing & Verification Checklist

```
Phase 3 — Core Services
[ ] Docker installed in CT101: docker --version
[ ] Portainer running: http://192.168.1.101:9000
[ ] Portainer admin account created
[ ] Vaultwarden running: http://192.168.1.101:8080
[ ] Vaultwarden admin account created
[ ] Vaultwarden USB backup done (encrypted JSON exported)
[ ] Rclone configured for B2: rclone lsd backblaze-b2:
[ ] B2 backup script created: /opt/vaultwarden/backup-to-b2.sh
[ ] B2 backup tested manually: script runs without error
[ ] Cron job set for daily backup at 2 AM
[ ] Immich running: http://192.168.1.101:2283 (3 containers: server, db, redis)
[ ] Immich admin account created
[ ] Stirling-PDF running: http://192.168.1.101:8081
[ ] Uptime Kuma running: http://192.168.1.101:3001
[ ] All service monitors added to Uptime Kuma
[ ] All proxy hosts added in Nginx PM
[ ] vault.yourdomain.com → Vaultwarden ✅
[ ] immich.yourdomain.com → Immich ✅
[ ] pdf.yourdomain.com → Stirling-PDF ✅
[ ] status.yourdomain.com → Uptime Kuma ✅
[ ] portainer.yourdomain.com → Portainer ✅
```

---

## Troubleshooting

### Problem: Immich shows only 2 containers running (missing one)

**Cause:** Either DB or Redis failed to start

**Solutions:**
```bash
cd /opt/immich
docker-compose logs immich-db
docker-compose logs immich-redis
# Fix the error shown, then:
docker-compose restart
```

If DB crashed: check disk space on CT101:
```bash
df -h /var/lib/docker
```

---

### Problem: Nginx PM shows 502 Bad Gateway via vault.yourdomain.com

**Cause:** Vaultwarden container not running, or Nginx PM pointing to wrong port

**Solutions:**
1. Verify Vaultwarden is running: `docker ps | grep vaultwarden`
2. Check Vaultwarden logs: `docker logs vaultwarden`
3. In Nginx PM → `vault.yourdomain.com` proxy host → verify Forward Port is `8080`
4. Test direct access: `http://192.168.1.101:8080` (bypassing Nginx)

---

### Problem: Immich ML containers high memory usage

**Cause:** Machine learning models loading into RAM

**Solutions:**
1. Immich Admin → **Administration** → **Machine Learning** → disable if not needed for now
2. Alternatively: reduce CT101 RAM allocation isn't recommended; 4 GB is already minimal

---

### Problem: rclone backup fails with "Access denied"

**Cause:** B2 Application Key permissions or wrong key configured

**Solutions:**
```bash
rclone lsd backblaze-b2:
# If this fails: re-run rclone config to re-enter credentials
rclone config show backblaze-b2
```

Verify the key has `listBuckets`, `listFiles`, `readFiles`, `writeFiles` capabilities in B2 dashboard.

---

### Problem: `*.yourdomain.com` URL loads but shows wrong service

**Cause:** pfSense host override pointing to wrong IP, or Nginx PM proxy host misconfigured

**Solutions:**
1. `nslookup vault.lan` → verify it returns `192.168.1.100` (Nginx PM, not CT101 directly)
2. In Nginx PM → `vault.yourdomain.com` proxy host → confirm Forward IP = `192.168.1.101`, Port = `8080`
3. In pfSense → DNS Resolver → Host Overrides → all service entries should have Domain = `yourdomain.com` and IP = `192.168.1.100`

---

### Problem: Docker out of disk space on CT101

**Cause:** Unused Docker images accumulating

**Solutions:**
```bash
docker system prune -f        # Remove stopped containers + unused images + networks
docker volume prune -f        # ⚠️ Only if volumes are truly unused
df -h /var/lib/docker         # Check space
```

---

## Next Steps

**Phase 3 complete!** All personal services are running and accessible via `.lan` domains.

**Phase 4 — Remote Access** *(guide not yet written)*:
- Set up Cloudflare Tunnel to expose selected services publicly
- Configure Cloudflare Access for zero-trust authentication
- Enable HTTPS with real certificates via Nginx PM + Let's Encrypt
- Access your services from outside your home network

**Phase 5 — Dev Environment & CI/CD** *(guide not yet written)*:
- Gitea (self-hosted Git)
- Jenkins + SonarQube
- Container registry

---

*Last updated: 2026-05-01*
*Based on: HP EliteDesk 800 G4 SFF (i5-8500, 16GB DDR4, 500GB NVMe)*
*CT101: 192.168.1.101 (4GB RAM, 50GB), Docker services*
*Immich: v1.91+ (pgvecto-rs, no Typesense)*
*B2 backup path: `backblaze-b2:homelab-backups/vaultwarden/`*
