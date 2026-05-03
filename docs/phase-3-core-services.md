# Homelab Setup Guide — Phase 3
## Core Services

**Target:** Deploy all personal services in CT101 via Docker — Portainer, Vaultwarden, Immich, Stirling-PDF, Uptime Kuma, Syncthing, Memos
**Timeframe:** Day 2–3 (4–6 hours)
**Prerequisite:** [Phase 2 — Network Services](phase-2-network-services.md) complete (CT101 running, Traefik ready at 192.168.1.100:443)
**Outcome:** All personal services accessible via `*.yourdomain.com` internal HTTPS URLs through Traefik
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
9. [Set Up Syncthing](#step-38-set-up-syncthing-file-sync)
10. [Set Up Memos](#step-39-set-up-memos-quick-notes)
11. [Configure Traefik Routes](#step-310-configure-traefik-routes)
12. [Test DNS & Access Services](#step-311-test-dns--access-services)
13. [Backup Strategy](#step-312-backup-strategy)
14. [Testing & Verification Checklist](#testing--verification-checklist)
15. [Troubleshooting](#troubleshooting)

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
| **Syncthing** | Docker | 8384 (internal) | CT101 | File sync — phone/devices to server |
| **Memos** | Docker | 5230 (internal) | CT101 | Quick notes (Google Keep replacement) |

> **AdGuard Home** is already running in CT100 (set up in Phase 2 Step 2.6 as a system service). No need to install it again here.

### URL Access Map (After Phase 3)

| URL | Service | Underlying Port |
|---|---|---|
| `https://vault.yourdomain.com` | Vaultwarden | CT101:8080 → Traefik |
| `https://adguard.yourdomain.com` | AdGuard Home | CT100:3000 → Traefik (same host, loopback) |
| `https://immich.yourdomain.com` | Immich | CT101:2283 → Traefik |
| `https://pdf.yourdomain.com` | Stirling-PDF | CT101:8081 → Traefik |
| `https://status.yourdomain.com` | Uptime Kuma | CT101:3001 → Traefik |
| `https://portainer.yourdomain.com` | Portainer | CT101:9000 → Traefik |
| `https://syncthing.yourdomain.com` | Syncthing | CT101:8384 → Traefik |
| `https://memos.yourdomain.com` | Memos | CT101:5230 → Traefik |

> **All `*.yourdomain.com` URLs route through Traefik at 192.168.1.100 — not directly to CT101. pfSense resolves them internally via split-DNS. All routes are HTTPS with auto-renewed Let's Encrypt certs via Cloudflare DNS challenge.**

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

> **Route via Traefik:** Once Traefik is running, add the `portainer` route to `/opt/traefik/config/services.yml` on CT100 (covered in Step 3.10).

---

### Step 3.3: Set Up Vaultwarden (Password Manager)

> **Goal:** Self-hosted password manager. Critical service — MUST be backed up to Backblaze B2.

#### 3.3.1: Create Docker Compose for Vaultwarden

```bash
mkdir -p /opt/vaultwarden
cd /opt/vaultwarden
```

**Step A — Install argon2 and generate the admin token hash:**

> **Why argon2id?** Vaultwarden ≥1.28.0 recommends hashing the admin token with argon2id. If someone reads your `.env` file (e.g. a backup leak), a raw token is immediately usable — an argon2id hash is not. This is the approach documented in the [Vaultwarden Wiki: Enabling admin page](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page).

```bash
apt install -y argon2
```

Choose a strong but **memorable** admin password — you will type this in the browser when accessing `/admin`. Do not use a random string; you need to remember it.

```bash
# Replace "YourStrongAdminPassword" with your chosen password
echo -n "YourStrongAdminPassword" | argon2 "$(openssl rand -base64 32)" -id -k 65540 -t 3 -p 4 -e
```

You will see output like:
```
$argon2id$v=19$m=65540,t=3,p=4$abc123...=$xyz789...=
```

Copy the entire `$argon2id$...` string — this is the **hash** you store, not the password itself.

**Step B — Create `.env` file for secrets (never committed to Git):**

> **Why `.env`?** Your `docker-compose.yml` is tracked in Git. Secrets inside a committed file are a credential leak — even in a private repo, a misconfigured repo setting or future fork exposes them. The `.env` file stays on disk only, with `chmod 600`. ([OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html))

```bash
cat > .env << 'EOF'
# Vaultwarden secrets — DO NOT commit this file to Git
# Single quotes are required to prevent $ signs in the argon2 hash from being misinterpreted
ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$paste_your_full_hash_here'
EOF

chmod 600 .env
```

Replace the placeholder with your actual `$argon2id$...` hash from Step A (keep the surrounding single quotes).

Create `/opt/vaultwarden/.env.example` (safe to commit — documents required variables without real values):

```bash
cat > .env.example << 'EOF'
# Vaultwarden admin panel token (argon2id hash — NOT the plaintext password)
# Generate with:
#   echo -n "YourAdminPassword" | argon2 "$(openssl rand -base64 32)" -id -k 65540 -t 3 -p 4 -e
# Then paste the full $argon2id$... output below (wrap in single quotes).
# Login at: https://vault.yourdomain.com/admin using your plaintext password.
ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$replace_this_with_your_hash'
EOF
```

**Step C — Add `.env` to `.gitignore`:**

```bash
echo ".env" >> .gitignore
echo "vw-data/" >> .gitignore
```

> `vw-data/` contains the SQLite database with all your passwords — it must never be committed. Backblaze B2 backup (Step 3.4) is its offsite copy.

**Step D — Create `docker-compose.yml` (no secrets inside):**

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
    env_file:
      - .env                        # ← secrets loaded from .env, not hardcoded here
    environment:
      DOMAIN: https://vault.yourdomain.com
      SIGNUPS_ALLOWED: "true"       # ← enabled for first-time account creation only
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

> Replace `yourdomain.com` with your actual domain. `docker-compose.yml` is safe to commit — it contains no secrets.

#### 3.3.2: Start Vaultwarden

```bash
docker-compose up -d
```

Wait ~10 seconds for startup.

#### 3.3.3: Verify Vaultwarden Running

```bash
docker ps | grep vaultwarden
```

**Expected:** A line showing `vaultwarden` and status `Up X seconds` ✅

> ⚠️ **Do NOT access Vaultwarden via `http://192.168.1.101:8080` directly.** Vaultwarden requires a secure context (HTTPS) for its Web Crypto API — opening it over plain HTTP will show the error *"You need to enable HTTPS!"* and the UI will not function.
>
> The correct URL is `https://vault.yourdomain.com` — this becomes available after Step 3.10 (Traefik routes). Continue to Step 3.3.4 for now and return to create your account once Traefik is configured.

#### 3.3.4: Create Your Account (First-Time Setup)

> **Do this after Step 3.10 (Traefik routes).** Vaultwarden only works over HTTPS.

**Step 1 — Create account via the web UI:**

1. Open `https://vault.yourdomain.com` in your browser
2. Click **Create account**
3. Fill in your email and a strong master password (min 12 chars, e.g. `VaultMaster@2026!Secure`)
4. Click **Create account**

You are now logged in ✅

**Step 2 — Disable public signups:**

Once your account is created, prevent anyone else from registering:

```bash
ssh root@192.168.1.101
cd /opt/vaultwarden
```

Edit `docker-compose.yml` and change `SIGNUPS_ALLOWED` from `"true"` to `"false"`:

```bash
sed -i 's/SIGNUPS_ALLOWED: "true"/SIGNUPS_ALLOWED: "false"/' docker-compose.yml
docker-compose up -d
```

Verify it took effect — the **Create account** link should no longer appear at `https://vault.yourdomain.com`.

**Step 3 — Verify admin panel access:**

The admin panel lets you manage users, check server status, and re-enable signups temporarily if you ever need to add another user:

```
https://vault.yourdomain.com/admin
```

Enter the **admin password** you chose in Step 3.3.1 (not the argon2 hash — the original plaintext password you typed). Vaultwarden verifies your input against the stored hash. You should see the Vaultwarden admin dashboard ✅

> The argon2 hash in `.env` is a one-way hash — Vaultwarden re-hashes what you type and compares. You never store or use the raw hash directly.

**Write down / store in a secure note:**
- Vaultwarden URL: `https://vault.yourdomain.com`
- Account email: your-email@gmail.com
- Master password: (your chosen password)
- Admin panel: `https://vault.yourdomain.com/admin` + your admin password (not the hash)

#### 3.3.5: Offline USB Backup (Emergency Recovery)

> **Why:** If Backblaze B2 is unavailable or the backup job fails, you need a physical backup accessible offline.

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
> **Tool:** restic (NOT rclone — rclone syncs and can overwrite good data with corrupted data; restic creates immutable snapshots)
> **Backend:** Backblaze B2 with Object Lock enabled (ransomware-proof, accidental-delete-proof)

#### 3.4.1: Create Backblaze B2 Account

1. Go to **backblaze.com** → **Sign up** → create account
2. Verify email

#### 3.4.2: Create B2 Bucket with Object Lock

1. B2 dashboard → **Buckets** → **Create a Bucket**
2. Bucket name: `homelab-backups`
3. **Files in Bucket:** Private
4. **Object Lock:** Enable → **Governance mode** (protects against accidental delete + ransomware; you can still unlock as account owner if needed)
5. Create bucket

> **Why Object Lock matters:** With Object Lock, even if an attacker steals your B2 API key and tries to delete all backups, B2 will refuse. Locked objects cannot be deleted or overwritten during the retention period.

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
```

**Save both values in Vaultwarden** (once set up) or in a secure note now.

#### 3.4.4: Install restic in Core Services Container

SSH into container 101:

```bash
apt update && apt install -y restic
restic version
```

#### 3.4.5: Configure B2 credentials as environment variables

```bash
cat > /etc/restic-b2.env << 'EOF'
export B2_ACCOUNT_ID=your_application_key_id_here
export B2_ACCOUNT_KEY=your_application_key_here
export RESTIC_PASSWORD=your_strong_restic_repo_password_here
EOF

chmod 600 /etc/restic-b2.env
```

Create `/opt/vaultwarden/restic-b2.env.example` (safe to commit — kept alongside the backup script for reference):

```bash
cat > /opt/vaultwarden/restic-b2.env.example << 'EOF'
# Backblaze B2 credentials for restic backup
# B2_ACCOUNT_ID: Application Key ID from B2 dashboard → Account → Application Keys
# B2_ACCOUNT_KEY: Application Key secret (shown once at creation)
# RESTIC_PASSWORD: Encryption key for the restic repo — store in Vaultwarden
#                  If lost, backups are permanently unreadable
export B2_ACCOUNT_ID=your_application_key_id_here
export B2_ACCOUNT_KEY=your_application_key_here
export RESTIC_PASSWORD=your_strong_restic_repo_password_here
EOF
```

> **RESTIC_PASSWORD** is the encryption key for your backup repository. Store it in Vaultwarden. If you lose it, the backups are permanently unreadable.

#### 3.4.6: Initialise the restic repository

```bash
source /etc/restic-b2.env
restic -r b2:homelab-backups:/vaultwarden init
```

**Expected output:**
```
created restic repository xxxxxxxx at b2:homelab-backups:/vaultwarden
Please note that knowledge of your password is required to access the repository.
Losing your password means that your data is irrecoverably lost!
```

#### 3.4.7: Create backup script

```bash
cat > /opt/vaultwarden/backup-to-b2.sh << 'SCRIPT'
#!/bin/bash
# Vaultwarden restic backup to Backblaze B2
# Tool: restic (immutable snapshots, always encrypted, deduplication)

set -euo pipefail
source /etc/restic-b2.env

REPO="b2:homelab-backups:/vaultwarden"

# 1. Backup data directory
restic -r "$REPO" backup /opt/vaultwarden/vw-data \
  --tag vaultwarden \
  --exclude '*.tmp'

# 2. Verify snapshot integrity
restic -r "$REPO" check --read-data-subset=5%

# 3. Prune: keep 30 daily, 12 monthly snapshots
restic -r "$REPO" forget \
  --keep-daily 30 \
  --keep-monthly 12 \
  --prune

echo "restic backup completed: $(date)" >> /var/log/vw-backup.log
SCRIPT

chmod +x /opt/vaultwarden/backup-to-b2.sh
```

#### 3.4.8: Schedule daily backup via cron

```bash
crontab -e
```

Add at the end:
```
0 2 * * * /opt/vaultwarden/backup-to-b2.sh >> /var/log/vw-backup.log 2>&1
```

Save: Ctrl+O → Enter → Ctrl+X

#### 3.4.9: Test backup manually and verify

```bash
# Run backup
/opt/vaultwarden/backup-to-b2.sh

# Verify snapshots exist
source /etc/restic-b2.env
restic -r b2:homelab-backups:/vaultwarden snapshots
# Must show at least one snapshot

# Test restore to temp dir (non-destructive verification)
restic -r b2:homelab-backups:/vaultwarden restore latest --target /tmp/vw-restore
ls /tmp/vw-restore/opt/vaultwarden/vw-data
# Must show db.sqlite3

rm -rf /tmp/vw-restore
```

✅ restic backup with Object Lock configured. Your passwords are now protected against corruption, ransomware, and accidental delete.

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

In Uptime Kuma dashboard → **Add New Monitor**.

For each row below: click **Add New Monitor** → set **Monitor Type** to `HTTP(s)` → fill in **Friendly Name** and **URL** → set **Heartbeat Interval** to `60` seconds → click **Save**.

| Friendly Name | Monitor Type | URL |
|---|---|---|
| Vaultwarden | HTTP(s) | `http://192.168.1.101:8080/alive` |
| AdGuard Home | HTTP(s) | `http://192.168.1.100:3000` |
| Immich | HTTP(s) | `http://192.168.1.101:2283/api/server/ping` |
| Stirling-PDF | HTTP(s) | `http://192.168.1.101:8081/health` |
| Syncthing | HTTP(s) | `http://192.168.1.101:8384` |
| Memos | HTTP(s) | `http://192.168.1.101:5230` |
| Traefik | HTTP(s) | `https://traefik.yourdomain.com` |
| Proxmox | HTTP(s) | `https://192.168.1.10:8006` |

**After adding all monitors, they should all show "UP" ✅**

> **Note:** Syncthing and Memos monitors will show "DOWN" until those services are deployed in Steps 3.8 and 3.9. That is expected — add them now and they will turn green after those steps.

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

### Step 3.8: Set Up Syncthing (File Sync)

> **Goal:** Sync files from your phone and other devices to the server automatically. Replaces Google Drive/iCloud for documents and photos. Uses P2P encrypted sync — no account required, works on your LAN without internet.
> **RAM usage:** ~50 MB idle.
> **Ports:** `8384` (web admin UI), `22000` (sync protocol), `21027` (device discovery).

#### 3.8.1: Create Docker Compose for Syncthing

SSH into container 101 (if not already connected):

```bash
ssh root@192.168.1.101
```

Create the directory and compose file:

```bash
mkdir -p /opt/syncthing
cd /opt/syncthing

cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  syncthing:
    image: lscr.io/linuxserver/syncthing:latest
    container_name: syncthing
    restart: always
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Jakarta
    ports:
      - "8384:8384"      # Web admin UI
      - "22000:22000/tcp" # Sync protocol (TCP)
      - "22000:22000/udp" # Sync protocol (QUIC/UDP)
      - "21027:21027/udp" # Local device discovery
    volumes:
      - ./config:/config
      - ./data:/data
    networks:
      - syncthing-network

networks:
  syncthing-network:
    driver: bridge
EOF
```

> **TZ=Asia/Jakarta:** Change this to your timezone if you are not in Indonesia (e.g. `Asia/Singapore`, `America/New_York`). This affects timestamps on sync logs.

#### 3.8.2: Start Syncthing

```bash
docker-compose up -d
```

Wait ~10 seconds for startup.

#### 3.8.3: Verify Syncthing Is Running

```bash
docker ps | grep syncthing
```

**Expected output:** A line showing `syncthing` and status `Up X seconds` ✅

#### 3.8.4: Access Syncthing Web UI

From your laptop browser:
```
http://192.168.1.101:8384
```

You will see the Syncthing dashboard. The first time you open it, it may prompt you to set a GUI password:

1. Click **Actions** (top-right) → **Settings**
2. Click the **GUI** tab
3. Set **GUI Authentication User** and **GUI Authentication Password**
4. Click **Save**

#### 3.8.5: Connect Your Android Phone

1. Install **Syncthing** on your phone:
   - F-Droid (recommended): search `Syncthing`
   - Google Play: search `Syncthing`
2. Open Syncthing on your phone → tap the menu → **Show device ID**
3. Copy the long alphanumeric device ID
4. On your laptop browser, go to `http://192.168.1.101:8384`
5. Click **Add Remote Device** → paste the device ID → give it a name (e.g. `My Phone`) → click **Save**
6. On your phone, a notification will appear asking to accept the connection → tap **Accept**

The two devices are now paired.

#### 3.8.6: Create a Shared Folder (e.g. Phone Camera Roll)

On the Syncthing web UI (`http://192.168.1.101:8384`):

1. Click **Add Folder**
2. **Folder Label:** `Phone Camera` (or any name you like)
3. **Folder Path:** `/data/phone-camera` (this maps to `./data/phone-camera` inside the container)
4. Under the **Sharing** tab: tick your phone's device name
5. Click **Save**

On your phone, a notification appears to accept the shared folder → tap **Accept** and choose a local folder (e.g. your DCIM folder).

Files will now sync automatically whenever phone and server are on the same network.

#### 3.8.7: Add pfSense DNS Override for Syncthing

1. Open pfSense web UI → **Services** → **DNS Resolver** → **Host Overrides**
2. Click **+ Add**:
   - **Host:** `syncthing`
   - **Domain:** `yourdomain.com`
   - **IP Address:** `192.168.1.100` ← Traefik (CT100), not CT101
   - **Description:** `Syncthing via Traefik`
3. Click **Save** → **Apply Changes**

The Traefik HTTPS route (`https://syncthing.yourdomain.com`) is configured in Step 3.10.

> ⚠️ **Security note:** Keep the Syncthing admin UI accessible on LAN only. Do not expose port 8384 to the internet. Traefik will serve it internally via HTTPS.

---

### Step 3.9: Set Up Memos (Quick Notes)

> **Goal:** Self-hosted quick note app — replaces Google Keep. Runs as a single lightweight container. Accessible from any browser on your LAN.
> **RAM usage:** ~50 MB.
> **Port 5230:** Web UI and REST API.

#### 3.9.1: Create Docker Compose for Memos

SSH into container 101 (if not already connected):

```bash
ssh root@192.168.1.101
```

Create the directory and compose file:

```bash
mkdir -p /opt/memos
cd /opt/memos

cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  memos:
    image: neosmemo/memos:stable
    container_name: memos
    restart: always
    ports:
      - "5230:5230"
    volumes:
      - ./data:/var/opt/memos
    networks:
      - memos-network

networks:
  memos-network:
    driver: bridge
EOF
```

#### 3.9.2: Start Memos

```bash
docker-compose up -d
```

Wait ~10 seconds for startup.

#### 3.9.3: Verify Memos Is Running

```bash
docker ps | grep memos
```

**Expected output:** A line showing `memos` and status `Up X seconds` ✅

#### 3.9.4: Access Memos Web UI

From your laptop browser:
```
http://192.168.1.101:5230
```

You will see the Memos welcome screen:

1. Click **Sign up**
2. Enter a username (e.g. your first name) and password
3. Click **Sign up**

You are now in Memos. You can start writing notes immediately. ✅

**On your phone:** Navigate to `http://192.168.1.101:5230` in your phone browser and add it to your home screen for quick access (works as a PWA — Progressive Web App).

#### 3.9.5: Add pfSense DNS Override for Memos

1. Open pfSense web UI → **Services** → **DNS Resolver** → **Host Overrides**
2. Click **+ Add**:
   - **Host:** `memos`
   - **Domain:** `yourdomain.com`
   - **IP Address:** `192.168.1.100` ← Traefik (CT100), not CT101
   - **Description:** `Memos via Traefik`
3. Click **Save** → **Apply Changes**

The Traefik HTTPS route (`https://memos.yourdomain.com`) is configured in Step 3.10.

---

## DNS & Reverse Proxy Configuration

### Step 3.10: Configure Traefik Routes

> **Goal:** Define HTTPS routes for all CT101 services in Traefik's file provider config on CT100.
> **Why file provider, not labels:** Traefik runs on CT100 and reads only its own Docker daemon. Labels on CT101 containers are on a completely separate Docker daemon — invisible to Traefik. The file provider (a YAML config file on disk) is the correct approach for cross-host routing.

#### 3.10.1: SSH into CT100 and populate services.yml

```bash
ssh root@192.168.1.100
```

Replace the placeholder `services.yml` created in Phase 2 with the full config for all CT101 services:

```bash
cat > /opt/traefik/config/services.yml << 'EOF'
# Traefik file provider — cross-host routes for CT101 services
# Traefik watches this file (watch=true) and hot-reloads on every save.
# Replace yourdomain.com with your actual domain throughout this file.
http:
  routers:
    vault:
      rule: "Host(`vault.yourdomain.com`)"
      entryPoints: [websecure]
      tls:
        certResolver: cloudflare
      service: vault-svc

    portainer:
      rule: "Host(`portainer.yourdomain.com`)"
      entryPoints: [websecure]
      tls:
        certResolver: cloudflare
      service: portainer-svc

    immich:
      rule: "Host(`immich.yourdomain.com`)"
      entryPoints: [websecure]
      tls:
        certResolver: cloudflare
      service: immich-svc

    pdf:
      rule: "Host(`pdf.yourdomain.com`)"
      entryPoints: [websecure]
      tls:
        certResolver: cloudflare
      service: pdf-svc

    status:
      rule: "Host(`status.yourdomain.com`)"
      entryPoints: [websecure]
      tls:
        certResolver: cloudflare
      service: status-svc

    syncthing:
      rule: "Host(`syncthing.yourdomain.com`)"
      entryPoints: [websecure]
      tls:
        certResolver: cloudflare
      service: syncthing-svc

    memos:
      rule: "Host(`memos.yourdomain.com`)"
      entryPoints: [websecure]
      tls:
        certResolver: cloudflare
      service: memos-svc

    adguard:
      rule: "Host(`adguard.yourdomain.com`)"
      entryPoints: [websecure]
      tls:
        certResolver: cloudflare
      service: adguard-svc

  services:
    vault-svc:
      loadBalancer:
        servers:
          - url: "http://192.168.1.101:8080"

    portainer-svc:
      loadBalancer:
        servers:
          - url: "http://192.168.1.101:9000"

    immich-svc:
      loadBalancer:
        servers:
          - url: "http://192.168.1.101:2283"

    pdf-svc:
      loadBalancer:
        servers:
          - url: "http://192.168.1.101:8081"

    status-svc:
      loadBalancer:
        servers:
          - url: "http://192.168.1.101:3001"

    syncthing-svc:
      loadBalancer:
        servers:
          - url: "http://192.168.1.101:8384"

    memos-svc:
      loadBalancer:
        servers:
          - url: "http://192.168.1.101:5230"

    adguard-svc:
      loadBalancer:
        servers:
          # AdGuard runs as a systemd service on CT100 — use CT100's LAN IP, not loopback
          - url: "http://192.168.1.100:3000"
EOF
```

> **Replace** `yourdomain.com` with your actual domain throughout the file above.

Routes go live within seconds — no Traefik restart needed (Traefik watches the file and hot-reloads automatically).

#### 3.10.2: Verify routes are active

Check that Traefik picked up the new config without errors:

```bash
docker logs traefik 2>&1 | tail -20
# Look for: "Configuration loaded" or "Adding route"
# No errors = file was parsed correctly
```

#### 3.10.3: Verify all routers appear in Traefik dashboard

From your laptop browser:
```
https://traefik.yourdomain.com
```

**Expected:** Traefik dashboard → **HTTP** → **Routers** → shows `vault`, `portainer`, `immich`, `pdf`, `status`, `syncthing`, `memos`, `adguard` → all green

> **Note:** The `adguard` router proxies to `http://192.168.1.100:3000` — AdGuard's web UI runs as a systemd service on CT100. Using the LAN IP (not `127.0.0.1`) avoids 502 Bad Gateway errors when Traefik resolves the backend. DNS on port 53 is not affected by Traefik.

If a router is missing or red: check `services.yml` YAML indentation. YAML requires exactly 2 spaces per indentation level — no tabs. Fix and re-save; Traefik will reload within seconds.

#### 3.10.4: How to add a future service

For any new service deployed to CT101, append to `services.yml` on CT100:

```yaml
# Under http.routers: (same indentation as existing routers)
    new-service:
      rule: "Host(`new-service.yourdomain.com`)"
      entryPoints: [websecure]
      tls:
        certResolver: cloudflare
      service: new-service-svc

# Under http.services: (same indentation as existing services)
    new-service-svc:
      loadBalancer:
        servers:
          - url: "http://192.168.1.101:PORT"
```

Also add a pfSense host override: **Services** → **DNS Resolver** → **Host Overrides** → `new-service` → `yourdomain.com` → `192.168.1.100`.

The CT101 `docker-compose.yml` for the new service needs **no Traefik labels and no proxy network** — the service just needs to listen on `192.168.1.101:PORT`.

---

### Step 3.11: Test DNS & Access Services

#### 3.11.1: Add Missing pfSense DNS Overrides

Before testing, confirm all services have a pfSense DNS override pointing at Traefik (`192.168.1.100`). Go to pfSense → **Services** → **DNS Resolver** → **Host Overrides** and verify the following entries exist:

| Host | Domain | IP Address |
|---|---|---|
| `vault` | `yourdomain.com` | `192.168.1.100` |
| `portainer` | `yourdomain.com` | `192.168.1.100` |
| `adguard` | `yourdomain.com` | `192.168.1.100` | ← already added in Phase 2 |
| `immich` | `yourdomain.com` | `192.168.1.100` |
| `pdf` | `yourdomain.com` | `192.168.1.100` |
| `status` | `yourdomain.com` | `192.168.1.100` |
| `syncthing` | `yourdomain.com` | `192.168.1.100` |
| `memos` | `yourdomain.com` | `192.168.1.100` |
| `traefik` | `yourdomain.com` | `192.168.1.100` | ← already added in Phase 2 |

Add any that are missing, then click **Apply Changes**.

#### 3.11.2: Verify DNS Resolution

From your laptop terminal:

```bash
nslookup vault.yourdomain.com
# Expected: Address 192.168.1.100 (pfSense split-DNS intercepted it — correct)

nslookup memos.yourdomain.com
# Expected: Address 192.168.1.100

nslookup syncthing.yourdomain.com
# Expected: Address 192.168.1.100
```

If any return a different IP or fail to resolve: the pfSense host override for that service is missing or has a typo. Re-check Step 3.11.1.

#### 3.11.3: Access All Services via HTTPS URLs

Test each URL from your laptop browser:

| URL | Expected Result |
|---|---|
| `https://vault.yourdomain.com` | Vaultwarden login page |
| `https://portainer.yourdomain.com` | Portainer login |
| `https://adguard.yourdomain.com` | AdGuard Home dashboard (proxied by Traefik → CT100:3000) |
| `https://immich.yourdomain.com` | Immich web UI |
| `https://pdf.yourdomain.com` | Stirling-PDF dashboard |
| `https://status.yourdomain.com` | Uptime Kuma dashboard |
| `https://syncthing.yourdomain.com` | Syncthing web UI |
| `https://memos.yourdomain.com` | Memos notes UI |
| `https://traefik.yourdomain.com` | Traefik dashboard |

All should load with a valid HTTPS certificate (padlock icon in browser — issued by Let's Encrypt via Cloudflare DNS challenge) ✅

If you see a certificate warning: the cert may still be generating (wait 2 minutes and retry). If it persists, check Traefik logs for ACME errors.

---

### Step 3.12: Backup Strategy

> **Summary:** What gets backed up, where, and how often.

#### Automated Backups (Backblaze B2)

| Service | Data | Frequency | Method |
|---|---|---|---|
| Vaultwarden | `vw-data/` | Daily at 2 AM | restic → B2 (Object Lock) |
| Immich | `library/` | Weekly (planned) | restic → B2 (Object Lock) |

**restic snapshot retention policy:** 30 daily + 12 monthly snapshots
Restic deduplicates across snapshots — only changed chunks are uploaded each run.

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

--- Portainer ---
[ ] Portainer running: http://192.168.1.101:9000
[ ] Portainer admin account created

--- Vaultwarden ---
[ ] Vaultwarden container running: docker ps | grep vaultwarden
[ ] Vaultwarden accessible via HTTPS: https://vault.yourdomain.com (requires Step 3.10 first)
[ ] Vaultwarden admin account created
[ ] Vaultwarden USB backup done (encrypted JSON exported to USB drive)

--- Backblaze B2 Backup ---
[ ] restic installed in CT101: restic version
[ ] B2 bucket created with Object Lock (Governance mode)
[ ] /etc/restic-b2.env created with chmod 600
[ ] restic repo initialised: restic -r b2:homelab-backups:/vaultwarden snapshots
[ ] Backup tested manually: /opt/vaultwarden/backup-to-b2.sh runs without error
[ ] Restore tested: /tmp/vw-restore/opt/vaultwarden/vw-data/db.sqlite3 exists
[ ] Cron job set for daily backup at 2 AM: crontab -l shows the entry

--- AdGuard Home (set up in Phase 2, already done) ---
[ ] AdGuard Home still running: http://192.168.1.100:3000
[ ] https://adguard.yourdomain.com resolves to 192.168.1.100 (pfSense override from Phase 2)

--- Immich ---
[ ] Immich running: http://192.168.1.101:2283 (3 containers: server, db, redis)
[ ] Immich admin account created
[ ] Storage warning understood: new photos only until HDD arrives

--- Stirling-PDF ---
[ ] Stirling-PDF running: http://192.168.1.101:8081

--- Uptime Kuma ---
[ ] Uptime Kuma running: http://192.168.1.101:3001
[ ] Admin account created
[ ] All 7 monitors added (Vaultwarden, AdGuard, Immich, Stirling-PDF, Syncthing, Memos, Traefik, Proxmox)
[ ] All monitors showing UP (Syncthing and Memos may show DOWN until Steps 3.8/3.9 complete)

--- Syncthing ---
[ ] Syncthing running: http://192.168.1.101:8384
[ ] GUI password set
[ ] Phone paired as remote device
[ ] At least one shared folder configured and syncing
[ ] pfSense DNS override added: syncthing.yourdomain.com → 192.168.1.100

--- Memos ---
[ ] Memos running: http://192.168.1.101:5230
[ ] Admin account created
[ ] pfSense DNS override added: memos.yourdomain.com → 192.168.1.100

--- Traefik Routes ---
[ ] /opt/traefik/config/services.yml on CT100 populated with all 8 routes (vault, portainer, immich, pdf, status, syncthing, memos, adguard)
[ ] Traefik dashboard shows all 8 routers green: https://traefik.yourdomain.com

--- DNS & HTTPS ---
[ ] nslookup vault.yourdomain.com returns 192.168.1.100
[ ] nslookup memos.yourdomain.com returns 192.168.1.100
[ ] nslookup syncthing.yourdomain.com returns 192.168.1.100
[ ] https://vault.yourdomain.com → Vaultwarden ✅ (valid HTTPS cert)
[ ] https://adguard.yourdomain.com → AdGuard Home ✅
[ ] https://immich.yourdomain.com → Immich ✅
[ ] https://pdf.yourdomain.com → Stirling-PDF ✅
[ ] https://status.yourdomain.com → Uptime Kuma ✅
[ ] https://portainer.yourdomain.com → Portainer ✅
[ ] https://syncthing.yourdomain.com → Syncthing ✅
[ ] https://memos.yourdomain.com → Memos ✅
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

### Problem: Traefik shows 502 Bad Gateway for a service URL

**Cause:** Service not running on CT101, wrong port in `services.yml`, or `services.yml` YAML syntax error

**Solutions:**
```bash
# 1. Verify the service is running on CT101
ssh root@192.168.1.101
docker ps | grep SERVICE_NAME

# 2. Test direct access (bypassing Traefik entirely)
curl http://192.168.1.101:PORT
# If this fails: the service itself is down, not a Traefik issue

# 3. Check services.yml on CT100 has the correct port
ssh root@192.168.1.100
cat /opt/traefik/config/services.yml | grep -A3 SERVICE_NAME

# 4. Check Traefik picked up the config (no YAML parse errors)
docker logs traefik 2>&1 | tail -30 | grep -i "error\|warn"

# 5. Validate YAML syntax
python3 -c "import yaml; yaml.safe_load(open('/opt/traefik/config/services.yml'))"
# No output = valid YAML
```

---

### Problem: Immich ML containers high memory usage

**Cause:** Machine learning models loading into RAM

**Solutions:**
1. Immich Admin → **Administration** → **Machine Learning** → disable if not needed for now
2. Alternatively: reduce CT101 RAM allocation isn't recommended; 4 GB is already minimal

---

### Problem: restic backup fails

**Cause:** Wrong credentials, B2 connectivity issue, or repo not initialised

**Solutions:**
```bash
source /etc/restic-b2.env

# Test B2 connectivity
restic -r b2:homelab-backups:/vaultwarden snapshots
# If this errors with "wrong password": RESTIC_PASSWORD in .env is wrong
# If this errors with "unauthorized": B2_ACCOUNT_ID or B2_ACCOUNT_KEY is wrong

# Re-check credentials in B2 dashboard:
# Account → Application Keys → verify key is still active

# If repo is missing (first run):
restic -r b2:homelab-backups:/vaultwarden init
```

---

### Problem: `*.yourdomain.com` URL loads but shows wrong service

**Cause:** pfSense host override pointing to wrong IP, or Traefik label has wrong router name/hostname

**Solutions:**
1. `nslookup vault.yourdomain.com` → verify it returns `192.168.1.100` (Traefik host, not CT101 directly)
2. Check Traefik dashboard → **HTTP** → **Routers** → verify `vault` router rule shows `Host(\`vault.yourdomain.com\`)`
3. Check `services.yml` on CT100: `cat /opt/traefik/config/services.yml | grep -A2 vault`
4. In pfSense → DNS Resolver → Host Overrides → all service entries should have Domain = `yourdomain.com` and IP = `192.168.1.100`

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

### Problem: Syncthing shows "Disconnected" for phone

**Cause:** Phone and server are not on the same network, or discovery is blocked

**Solutions:**
```bash
# 1. Confirm Syncthing container is running
docker ps | grep syncthing

# 2. Check all ports are open (should see 8384, 22000, 21027)
ss -tulnp | grep -E "8384|22000|21027"

# 3. In Syncthing web UI → Actions → Advanced → confirm the device ID matches what you entered on your phone

# 4. Try adding the server's IP manually on your phone:
#    Syncthing app → (server device) → Edit → Addresses → add tcp://192.168.1.101:22000
```

---

### Problem: Memos container starts but UI shows blank page or error

**Cause:** Data directory permissions issue or port conflict

**Solutions:**
```bash
cd /opt/memos

# Check container logs for errors
docker-compose logs memos

# Check if port 5230 is already used by something else
ss -tulnp | grep 5230

# If there is a permissions error on ./data, fix it:
chown -R 1000:1000 ./data
docker-compose restart
```

---

## Next Steps

**Phase 3 complete!** All personal services are running and accessible via `*.yourdomain.com` HTTPS URLs through Traefik.

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

*Last updated: 2026-05-03*
*Based on: HP EliteDesk 800 G4 SFF (i5-8500, 16GB DDR4, 500GB NVMe)*
*CT101: 192.168.1.101 (4GB RAM, 50GB), Docker services*
*Immich: v1.91+ (pgvecto-rs, no Typesense)*
*B2 backup path: `b2:homelab-backups/vaultwarden/`*
*New in this revision: Syncthing (step 3.8), Memos (step 3.9); AdGuard Home moved to Phase 2 (CT100 system service)*
