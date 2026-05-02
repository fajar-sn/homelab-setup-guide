# Homelab Setup Guide — Phase 1
## Proxmox Installation & Foundation

**Target:** Install Proxmox VE on NVMe, configure networking, updates, and storage
**Timeframe:** Day 1–2 (2–3 hours)
**Prerequisite:** [Phase 0 — Preparation](phase-0-preparation.md) complete (USB installer ready, VT-x enabled)
**Outcome:** Proxmox running at `https://192.168.1.10:8006`, network bridge ready for LXC containers
**Next:** [Phase 2 — Network Services](phase-2-network-services.md)

---

## Table of Contents

1. [Boot Proxmox Installer](#step-11-boot-proxmox-installer)
2. [Proxmox Installation Wizard](#step-12-proxmox-installation-wizard)
3. [Post-Installation Boot](#step-13-post-installation-boot)
4. [First Login via Console](#step-14-first-login-via-console)
5. [Access Proxmox Web Dashboard](#step-15-access-proxmox-web-dashboard)
6. [Update Proxmox & Remove Subscription Notice](#step-16-update-proxmox--remove-subscription-notice)
7. [Configure Proxmox Backup Storage](#step-17-configure-proxmox-backup-storage)
8. [Configure Networking for LXC Containers](#step-18-configure-networking-for-lxc-containers)
9. [Enable HTTPS Certificate (Optional)](#step-19-enable-https-certificate-optional-but-recommended)
10. [Proxmox CLI Essentials](#step-110-proxmox-cli-essentials)
11. [Testing & Verification Checklist](#testing--verification-checklist)
12. [Troubleshooting](#troubleshooting)

---

### Step 1.1: Boot Proxmox Installer

#### 1.1.1: Insert USB & Power On

1. Insert **bootable USB** into HP EliteDesk front USB 3.0 port
2. Power on the machine
3. Proxmox boot menu appears (within 5 seconds)

#### 1.1.2: Select Installation Mode

```
Proxmox VE installer boot screen shows:
  ┌──────────────────────────────────────────────────────┐
  │ Proxmox Virtual Environment 9.1-1                    │
  ├──────────────────────────────────────────────────────┤
  │ Install Proxmox VE (Graphical)                       │ ← Select this
  │ Install Proxmox VE (Terminal UI)                     │
  │ Advanced Options: Install Proxmox VE (Graphical)     │
  │ Advanced Options: Install Proxmox VE (Terminal UI)   │
  │ Advanced Options: Rescue Boot                        │
  │ Advanced Options: Test Memory (memtest86+)            │
  └──────────────────────────────────────────────────────┘
```

**Action:** Use arrow keys to highlight **Install Proxmox VE (Graphical)** → Press **Enter**

> **Tip:** If the graphical installer fails to render, reboot and use **Install Proxmox VE (Terminal UI)** instead — it uses the same steps but text-only.

The installer loads Linux kernel + Proxmox installer (takes ~1 minute)

---

### Step 1.2: Proxmox Installation Wizard

#### 1.2.1: Welcome Screen — EULA

```
┌─────────────────────────────────────────┐
│  Proxmox Virtual Environment installer  │
│                                         │
│  Welcome to Proxmox VE 9.1-1            │
│  End User License Agreement (EULA)      │
│                                         │
│  [I agree] [Abort]                      │
└─────────────────────────────────────────┘
```

**Action:** Read the EULA (AGPL v3 license) then click **I agree**

---

#### 1.2.2: Target Hard Disk Selection

**This is the most critical step — you're selecting which disk to install to.**

```
┌──────────────────────────────────────────────────────────┐
│  Target Hard Disk                                        │
│                                                          │
│  [✓] nvme0n1 - 500.1 GB (WD Blue SN570)  [selected]    │
│  [ ] ata0    - (not used)                               │
│                                                          │
│  Options...  [Next]                                      │
└──────────────────────────────────────────────────────────┘
```

**Verify before proceeding:**
- [ ] **nvme0n1** is selected (shows your NVMe size, e.g., 500.1 GB)
- [ ] ⚠️ Any USB drives are NOT selected

**Click "Options..." to verify file system:**
- File system: **ext4** (default, fine for NVMe homelab)
- SSD discard: ☑ Enable (allows NVMe TRIM — extends drive life)
- Leave other options as default

**Action:** Click **Next**

---

#### 1.2.3: Location & Time

```
┌──────────────────────────────────────────────────────────┐
│  Location & Time Zone                                    │
│                                                          │
│  Country:    [Indonesia ▼]                              │
│  Time Zone:  [Asia/Jakarta ▼]                           │
│                                                          │
│  [Next]                                                  │
└──────────────────────────────────────────────────────────┘
```

**Action:** 
- Country: **Indonesia**
- Time Zone: **Asia/Jakarta**
- Click **Next**

---

#### 1.2.4: Network Configuration

**This is where you configure Proxmox to connect to your home LAN through pfSense.**

```
┌──────────────────────────────────────────────────────────┐
│  Network Configuration                                   │
│                                                          │
│  Hostname:           [homelab.yourdomain.com]            │
│  IP Address (CIDR):  [192.168.1.10/24]                  │
│  Gateway:            [192.168.1.1]                      │
│  DNS 1:              [8.8.8.8]                          │
│  DNS 2:              [8.8.4.4]                          │
│                                                          │
│  [Next]                                                  │
└──────────────────────────────────────────────────────────┘
```

**Explanation of each field:**

| Field | Your Value | Explanation |
|---|---|---|
| **Hostname** | `homelab.yourdomain.com` | FQDN of your Proxmox machine. Using a real domain enables Let's Encrypt SSL certificates later (Phase 1.9). |
| **IP Address** | `192.168.1.10/24` | Static IP on your pfSense LAN, `/24` = netmask 255.255.255.0 |
| **Gateway** | `192.168.1.1` | pfSense's LAN IP (default router for your LAN) |
| **DNS 1** | `8.8.8.8` | Google Public DNS (change later to AdGuard Home in Phase 2) |
| **DNS 2** | `8.8.4.4` | Google Public DNS (fallback) |

**Action:**
1. Enter exactly as shown above
2. Click **Next**

---

#### 1.2.5: Email & Password (Root Account)

```
┌──────────────────────────────────────────────────────────┐
│  Administrator Account                                   │
│                                                          │
│  Email Address:  [your-email@gmail.com]                 │
│  Password:       [••••••••••••] ← strong password!      │
│  Confirm:        [••••••••••••]                          │
│                                                          │
│  [Next]                                                  │
└──────────────────────────────────────────────────────────┘
```

**Important:**
- **Email:** Use a personal email (alerts will be sent here)
- **Password:** Must be 8+ characters, mix of upper/lower/numbers/special chars
  
**Example strong password:** `Proxmox@2026!HomeLabV1`

**⚠️ Save this password immediately** (Vaultwarden will manage it later, but note it down now)

**Action:**
1. Enter email + strong password
2. Click **Next**

---

#### 1.2.6: Installation Summary

```
┌──────────────────────────────────────────────────────────┐
│  Summary                                                 │
│                                                          │
│  Target Disk:    nvme0n1 (500.1 GB)                      │
│  File System:    ext4                                    │
│  Hostname:       homelab.yourdomain.com                  │
│  IP Address:     192.168.1.10/24                         │
│  Gateway:        192.168.1.1                             │
│  DNS:            8.8.8.8, 8.8.4.4                        │
│  Email:          your-email@gmail.com                    │
│                                                          │
│  ⚠️  All data on nvme0n1 will be ERASED                 │
│                                                          │
│  [Install] [Cancel]                                      │
└──────────────────────────────────────────────────────────┘
```

**⚠️ This is your last chance to cancel before data is erased.**

**Verification checklist:**
- [ ] Disk: **nvme0n1** (your NVMe SSD)
- [ ] Hostname: **homelab.yourdomain.com**
- [ ] IP: **192.168.1.10/24**
- [ ] Gateway: **192.168.1.1** (your pfSense IP)

**Action:** Click **Install**

**Installation begins** → takes ~3–5 minutes (progress bar shows extraction + formatting)

---

### Step 1.3: Post-Installation Boot

#### 1.3.1: Wait for Installation to Complete

```
┌──────────────────────────────────────────────────────────┐
│  Installing Proxmox VE                                   │
│                                                          │
│  ▓▓▓▓▓▓▓▓▓▓▓▓░░░░░ 65%                                  │
│                                                          │
│  Extracting files...                                     │
└──────────────────────────────────────────────────────────┘
```

**Wait 3–5 minutes until:**
```
Installation complete!

Remove USB drive and press any key to reboot.
```

#### 1.3.2: Eject USB & Reboot

1. **Power off** the HP EliteDesk (via on-screen prompt or button)
2. **Remove the USB drive**
3. Power on the machine
4. Proxmox boots from NVMe (no more installer)

**Boot sequence:**
```
BIOS ↓
GRUB (Proxmox bootloader) ↓
Linux kernel loads ↓
Proxmox services start ↓
(takes ~1 minute after POST)

Ready for login!
```

---

### Step 1.4: First Login via Console

> **Goal:** Verify Proxmox is running + initial system checks.

#### 1.4.1: Wait for Login Prompt

After boot completes (when you see):
```
Proxmox Virtual Environment 9.1-1
homelab login:
```

#### 1.4.2: Log in as Root

```
homelab login: root
Password: [the password you set in Step 1.2.5]
```

**Successful login shows:**
```
root@homelab:~#
```

#### 1.4.3: Verify Network Connectivity

```bash
# Test IP address assignment
ip addr show | grep 192.168

# Expected output:
# inet 192.168.1.10/24 brd 192.168.1.255 scope global ...

# Test DNS resolution
nslookup google.com

# Expected output:
# Server: 8.8.8.8
# Name: google.com
# Address: X.X.X.X
```

#### 1.4.4: Verify Proxmox Services

```bash
# Check Proxmox daemon status
systemctl status pveproxy

# Expected output:
# ● pveproxy.service - PVE API Proxy Server
#    Loaded: loaded
#    Active: active (running)
```

#### 1.4.5: Check NVMe Installation

```bash
# Verify disk is installed on correct drive
lsblk

# Expected output:
# NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
# nvme0n1     259:0    0 500.1G  0 disk
# ├─nvme0n1p1 259:1    0  1024M  0 part /boot/efi
# ├─nvme0n1p2 259:2    0  1024M  0 part /boot
# └─nvme0n1p3 259:3    0 497.1G  0 part /
```

#### 1.4.6: Verify Virtualization is Enabled

```bash
# Check for VMX support (Intel VT-x)
grep -o 'vmx' /proc/cpuinfo | head -1

# Expected output:
# vmx ✅

# If no output: VT-x is NOT enabled → go back to Step 0.2
```

#### 1.4.7: Verify `/etc/hosts` Entry

The installer automatically creates the correct `/etc/hosts` entry when you use an FQDN hostname. Verify:

```bash
cat /etc/hosts
```

**Expected:**
```
127.0.0.1 localhost.localdomain localhost
192.168.1.10 homelab.yourdomain.com homelab

::1     ip6-localhost ip6-loopback
...
```

> **If the entry is missing or wrong:** Edit manually with `nano /etc/hosts` and add:
> `192.168.1.10 homelab.yourdomain.com homelab`
> (Replace `192.168.1.10` with your actual Proxmox IP.)

**If everything shows ✅**, you can now access Proxmox via web UI.

---

### Step 1.5: Access Proxmox Web Dashboard

> **Goal:** Connect to Proxmox UI from your laptop to complete initial setup.

#### 1.5.1: From Your Laptop

1. Open a web browser (Firefox, Chrome, Edge, Safari)
2. Navigate to:
   ```
   https://192.168.1.10:8006
   ```

3. You'll see:
   ```
   ⚠️ Your connection is not private
   (Self-signed certificate warning)
   ```

   **This is normal.** Proxmox uses self-signed SSL certificates by default.

#### 1.5.2: Bypass Self-Signed Certificate Warning

**Firefox:**
1. Click **Advanced** (or similar)
2. Click **Accept the Risk and Continue**

**Chrome:**
1. Click **Advanced**
2. Click **Proceed to 192.168.1.10 (unsafe)**

#### 1.5.3: Proxmox Login Page

```
┌──────────────────────────────────────┐
│  Proxmox Virtual Environment         │
│                                      │
│  Username:  [root]                   │
│  Password:  [••••••••••••]           │
│  Realm:     [PAM]                    │
│                                      │
│  [Login]                             │
└──────────────────────────────────────┘
```

**Action:**
- Username: `root`
- Password: (the one you set in Step 1.2.5)
- Realm: `PAM` (keep as default)
- Click **Login**

#### 1.5.4: Proxmox Dashboard Appears

```
┌─────────────────────────────────────────────────────────┐
│  Proxmox Virtual Environment                            │
│                                                         │
│  Left Sidebar:                                          │
│  ├─ homelab (cluster)                                   │
│  │  ├─ homelab (node)                                   │
│  │  │  ├─ Summary                                       │
│  │  │  ├─ Nodes                                         │
│  │  │  ├─ Storage                                       │
│  │  │  └─ ...                                           │
│  ├─ Datacenter                                          │
│  └─ Help                                                │
│                                                         │
│  Center pane: System stats                              │
│  - CPU: 6C/6T @ 3.0GHz                                  │
│  - RAM: 16 GB                                           │
│  - Disk: 500 GB NVMe                                    │
│                                                         │
│  ✅ Welcome to Proxmox!                                 │
└─────────────────────────────────────────────────────────┘
```

**Congratulations! Proxmox is installed and accessible.** ✅

---

### Step 1.6: Update Proxmox & Remove Subscription Notice

> **Goal:** Install security updates + remove the "no subscription" warning from the dashboard.

#### 1.6.1: Update Package Lists

Click **Nodes** → **homelab** → **Shell** (in the right panel)

A terminal window opens. Type:

```bash
apt update
```

Output shows package lists being downloaded (~1 minute)

#### 1.6.2: Install System Updates

```bash
apt dist-upgrade -y
```

Output shows kernel + system packages being updated (~2–5 minutes depending on pending updates)

#### 1.6.3: Disable Subscription-Only Repositories

Proxmox VE 9 ships with **two** enterprise-only repos that both require a paid subscription — the PVE repo and the Ceph repo. Both cause `401 Unauthorized` errors on `apt update` if you don't have a subscription.

**Easiest method — Proxmox Web UI:**
1. In Proxmox web UI → **Nodes** → **homelab** → **Updates** → **Repositories**
2. Click `pve-enterprise` → **Disable**
3. Click `ceph-enterprise` → **Disable**

**CLI method — overwrite both files with correct content:**

```bash
cat > /etc/apt/sources.list.d/pve-entreprise.sources << 'EOF'
Types: deb
URIs: https://enterprise.proxmox.com/debian/pve
Suites: trixie
Components: pve-enterprise
Enabled: no
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

cat > /etc/apt/sources.list.d/ceph.sources << 'EOF'
Types: deb
URIs: https://enterprise.proxmox.com/debian/ceph-squid
Suites: trixie
Components: enterprise
Enabled: no
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

> **Note the spelling:** Proxmox uses the French spelling `pve-entreprise.sources` — not `pve-enterprise`. The *content* inside the file still uses `pve-enterprise` as the component name.

> **Why overwrite instead of sed?** deb822 `.sources` files are stanza-based and easy to corrupt with incremental edits. Overwriting with known-good content is idempotent and safe to re-run.

**Verify both are disabled:**
```bash
grep 'Enabled' /etc/apt/sources.list.d/pve-entreprise.sources /etc/apt/sources.list.d/ceph.sources
```

Expected output:
```
/etc/apt/sources.list.d/pve-entreprise.sources:Enabled: no
/etc/apt/sources.list.d/ceph.sources:Enabled: no
```

#### 1.6.4: Add No-Subscription Repository

Add the free community repository (Proxmox VE 9 / Debian 13 Trixie):

```bash
cat > /etc/apt/sources.list.d/pve-no-subscription.sources << 'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

> **Note:** Proxmox VE 9 is based on Debian 13 (Trixie). Earlier versions used `bullseye` or `bookworm`. Always verify the suite name at proxmox.com/wiki.

#### 1.6.5: Update Again

```bash
apt update
```

(Slower this time — fetching from Proxmox community repo)

#### 1.6.6: Remove Subscription Warning

The warning is a JavaScript check in the web UI. First, find the exact line in your installed version:

```bash
python3 - << 'EOF'
f = '/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js'
for i, line in enumerate(open(f), 1):
    if 'active' in line.lower() and ('status' in line.lower() or 'subscription' in line.lower()):
        print(f"Line {i}: {line.rstrip()[:200]}")
EOF
```

This prints the relevant line(s). Then patch using the exact text found:

```bash
python3 - << 'EOF'
import re, shutil, os
f = '/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js'
bak = f + '.bak'
# Always patch from the original backup so the script is safe to re-run
src = bak if os.path.exists(bak) else f
if not os.path.exists(bak):
    shutil.copy(f, bak)
txt = open(src).read()

# Covers all known PVE 7/8/9 variants (full null-check chain, standalone
# res.data.status check, and the older data.status form — with or without
# spaces / toLowerCase()).
patched = re.sub(
    r"(?:"
    r"res\s*===\s*null\s*\|\|\s*res\s*===\s*undefined\s*\|\|\s*!res\s*\|\|\s*res\.data\.status\.toLowerCase\(\)\s*!==\s*'active'"
    r"|res\.data\.status\.toLowerCase\(\)\s*!==\s*'active'"
    r"|res\.data\.status\s*!==\s*'active'"
    r"|data\.status\s*!==\s*'Active'"
    r"|data\.status\s*!==\s*'active'"
    r")",
    'false',
    txt
)

remaining = patched.count("!== 'active'") + patched.count("!== 'Active'")
open(f, 'w').write(patched)
print(f"Remaining: {remaining}")
EOF
```

Expected output:
```
Remaining: 0
```

> If `Remaining` is still non-zero, run the diagnostic above and share the printed line — there is a new pattern variant in your version that needs to be added.

#### 1.6.7: Restart Web UI Service

```bash
systemctl restart pveproxy
```

**Close your browser tab** and wait 10 seconds.

#### 1.6.8: Refresh Proxmox Dashboard

1. Go back to `https://192.168.1.10:8006`
2. Login again (credentials same as before)
3. ✅ Subscription warning is **gone**

---

### Step 1.7: Configure Proxmox Backup Storage

> **Goal:** Set up a local backup directory on NVMe so LXC snapshots + system backups have somewhere to go.

#### 1.7.1: Create Backup Directory

Back in the **Shell** (or open a new one), create a backup directory:

```bash
mkdir -p /var/lib/vz/dump/backups
chmod 700 /var/lib/vz/dump/backups
```

#### 1.7.2: Create Storage Configuration in Proxmox UI

1. In Proxmox web UI, left sidebar → **Datacenter** → **Storage**

2. Click **Add** → **Directory**

```
┌───────────────────────────────────────┐
│  Add Storage — Directory              │
│                                       │
│  ID:           [proxmox-local-backup] │
│  Directory:    [/var/lib/vz/dump]     │
│  Content:      [Disk image]           │
│                [VZDump backup file]   │
│  Nodes:        [homelab] (selected)   │
│  Disable:      [ ] (unchecked)        │
│                                       │
│  [Create]                             │
└───────────────────────────────────────┘
```

**Fill in:**
- **ID:** `proxmox-local-backup` (internal name)
- **Directory:** `/var/lib/vz/dump` (full path)
- **Content:** Check both:
  - ☑ Disk image
  - ☑ VZDump backup file
- **Nodes:** `homelab` should be checked

**Action:** Click **Create**

#### 1.7.3: Verify Storage is Available

Back in **Datacenter** → **Storage**, you should see:

```
┌─────────────────────────────────┐
│ Storage List                    │
├─────────────────────────────────┤
│ local         ext4   (system)   │
│ local-lvm     LVM    (VMs)      │
│ proxmox-local-backup DIR (✓)    │
└─────────────────────────────────┘
```

✅ Backup storage is ready.

---

### Step 1.8: Configure Networking for LXC Containers

> **Goal:** Set up a bridge so LXC containers can connect to your pfSense LAN.
> 
> **Key concept:** By default, Proxmox has `vmbr0` (VM bridge). We want LXCs on your home LAN (192.168.1.x), not isolated.

#### 1.8.1: Discover Your Physical Network Interface Name

Network interface names vary by hardware. **Never hardcode a name before checking.** Find yours first:

```bash
ip -brief link show | grep -vE '^(lo |vmbr|veth|fwbr|fwln|fwpr)'
```

**Example output:**
```
nic0             UP             c8:d9:d2:29:bc:a9
```

The first word is your NIC name. Common examples:
- `nic0` (Proxmox-renamed NIC — seen on HP EliteDesk 800 G4 with PVE 9)
- `eno1` (onboard NIC, older naming)
- `enp3s0` (PCI bus 3, slot 0)
- `eth0` (legacy naming)

**On HP EliteDesk 800 G4 with Proxmox VE 9, the name is typically `nic0`.**

> **If output is empty:** Your NIC is already enslaved to `vmbr0` (bridge is already configured). Verify with `bridge link` — if your NIC appears there, skip to Step 1.8.5.

> **Write this name down.** You'll use it in the next step.

#### 1.8.2: Understand Current Network Config

In Proxmox shell:

```bash
cat /etc/network/interfaces
```

Expected output (NIC name may differ):
```
auto lo
iface lo inet loopback

auto nic0            # ← your NIC name here
iface nic0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

**Explanation:**
- `nic0` = physical Ethernet port (your name may differ)
- `vmbr0` = virtual bridge (VMs/LXCs connect here)
- Currently both have IP `192.168.1.10` (redundant — the installer did this)

#### 1.8.3: Fix Network Configuration

We need to move the IP from the physical port to the bridge, so containers can access the LAN.

> **⚠️ Replace `nic0` below with your actual NIC name from Step 1.8.1.**

**Recommended: Use Proxmox Web UI (safer, no SSH drop):**
1. In Proxmox web UI → **Nodes** → **homelab** → **Network**
2. Click `vmbr0` → **Edit**:
   - Set **Bridge ports** to your NIC name (e.g., `nic0`)
   - Keep IP: `192.168.1.10/24`, Gateway: `192.168.1.1`
3. Click `nic0` → **Edit** → change to `No IP address (manual)`
4. Click **Apply Configuration** (top of page)

**Alternative: CLI** (may briefly drop SSH):

```bash
# Replace nic0 with your actual NIC name from Step 1.8.1
NIC=nic0

cat > /etc/network/interfaces << EOF
auto lo
iface lo inet loopback

auto ${NIC}
iface ${NIC} inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
    bridge-ports ${NIC}
    bridge-stp off
    bridge-fd 0
EOF
```

**Verify the file:**

```bash
cat /etc/network/interfaces
```

#### 1.8.4: Apply Network Changes

**If using CLI:**
```bash
ifreload -a
```

> Proxmox VE 9 uses `ifupdown2` by default (installed with Proxmox). `ifreload -a` applies changes live without a full restart — SSH connection stays up.

If `ifreload` is not available, use:
```bash
systemctl restart networking
```
(**This will drop SSH briefly.** Wait 10 seconds, then reconnect.)

#### 1.8.5: Verify Network Bridge

```bash
ip addr show vmbr0
```

Expected output:
```
4: vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 192.168.1.10/24 brd 192.168.1.255 scope global vmbr0
```

```bash
bridge link
```

Should show your NIC (e.g., `nic0`) attached to `vmbr0`.

✅ Network bridge is active and ready for LXC containers.

---

### Step 1.9: Enable HTTPS Certificate (Optional but Recommended)

> **Goal:** Replace the self-signed certificate with a Let's Encrypt certificate so you don't get warnings when accessing Proxmox.
>
> **Requirement:** Your domain must point to Proxmox's IP (192.168.1.10). This works only if:
> - You're on your home network
> - You've set up DNS on your internal network (which we do in Phase 2 with AdGuard Home)
>
> **For now:** Skip this step. We'll set it up properly after Phase 2 when AdGuard Home is configured.

---

### Step 1.10: Proxmox CLI Essentials

> **Goal:** Learn basic Proxmox commands for managing LXCs from the terminal (useful for automation later).

#### 1.10.1: List All Containers & VMs

```bash
pct list      # LXC containers
qm list       # QEMU VMs
```

#### 1.10.2: Get Detailed Container Info

```bash
pct status <container-id>
pct config <container-id>
```

#### 1.10.3: Start/Stop/Restart Container

```bash
pct start 100     # Start container 100
pct stop 100      # Stop container 100
pct reboot 100    # Reboot container 100
```

#### 1.10.4: Enter Container Shell

```bash
pct enter 100     # Enter container 100 as root
# Type 'exit' to leave
```

#### 1.10.5: View Container Logs

```bash
journalctl -u pve-container@100 -f
```

#### 1.10.6: Backup a Container

```bash
vzdump 100 --compress gzip --storage proxmox-local-backup
```

#### 1.10.7: Destroy a Container (Careful!)

```bash
pct destroy 100 --purge
```

---

## Testing & Verification Checklist

After completing all Phase 1 steps, verify:

```
[ ] Proxmox boots without USB drive
[ ] Proxmox web UI accessible at https://192.168.1.10:8006
[ ] Login with root works
[ ] No "subscription" warning banner (removed in Step 1.6.6)
[ ] apt update runs without errors
[ ] lsblk shows NVMe correctly (nvme0n1 with 3 partitions)
[ ] VT-x confirmed: grep -o vmx /proc/cpuinfo | head -1 shows "vmx"
[ ] /etc/hosts has: 192.168.1.10 homelab.yourdomain.com homelab
[ ] vmbr0 bridge has IP 192.168.1.10/24
[ ] bridge link shows NIC attached to vmbr0
[ ] Backup storage visible in Datacenter → Storage
[ ] System time correct (Jakarta timezone)
```

---

## Troubleshooting

### Problem: "No bootable device found"

**Cause:** BIOS not seeing USB, or USB not flashed correctly

**Solutions:**
1. Re-flash USB with Balena Etcher (use a different USB if possible)
2. In BIOS → Boot Options → check USB is listed
3. Try different USB port (rear USB ports are more reliable)
4. Try **Legacy Boot** mode (disable Secure Boot in BIOS)

---

### Problem: Proxmox web UI not accessible after install

**Cause:** Network misconfiguration during install

**Solutions:**
1. Console login → `ip addr show` → verify IP is 192.168.1.10
2. If wrong IP: `nano /etc/network/interfaces` → fix → `ifreload -a`
3. Verify pfSense DHCP range doesn't conflict (192.168.1.10 should be outside DHCP pool)
4. Check pfSense firewall — LAN rule should allow all traffic

---

### Problem: "subscription" warning still shows after Step 1.6.6

**Cause:** The `sed` command silently fails due to bash treating `!` as history expansion inside double quotes. The JS file is unmodified.

**Solution — use python3 instead:**
```bash
python3 - << 'EOF'
import re, shutil, os
f = '/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js'
bak = f + '.bak'
src = bak if os.path.exists(bak) else f
if not os.path.exists(bak):
    shutil.copy(f, bak)
txt = open(src).read()
patched = re.sub(
    r"(?:"
    r"res\s*===\s*null\s*\|\|\s*res\s*===\s*undefined\s*\|\|\s*!res\s*\|\|\s*res\.data\.status\.toLowerCase\(\)\s*!==\s*'active'"
    r"|res\.data\.status\.toLowerCase\(\)\s*!==\s*'active'"
    r"|res\.data\.status\s*!==\s*'active'"
    r"|data\.status\s*!==\s*'Active'"
    r"|data\.status\s*!==\s*'active'"
    r")",
    'false', txt
)
remaining = patched.count("!== 'active'") + patched.count("!== 'Active'")
open(f, 'w').write(patched)
print(f"Remaining: {remaining}")
EOF
systemctl restart pveproxy
```

---

### Problem: `apt update` fails with `401 Unauthorized` or `Malformed stanza` from enterprise.proxmox.com

**Cause:** Proxmox ships with two enterprise-only repos (`pve-entreprise.sources` and `ceph.sources`) enabled by default. Previous attempts to disable them with `echo >>` or `sed` may have also left the files in a corrupted state.

**Solution — overwrite both files with known-good content:**
```bash
cat > /etc/apt/sources.list.d/pve-entreprise.sources << 'EOF'
Types: deb
URIs: https://enterprise.proxmox.com/debian/pve
Suites: trixie
Components: pve-enterprise
Enabled: no
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

cat > /etc/apt/sources.list.d/ceph.sources << 'EOF'
Types: deb
URIs: https://enterprise.proxmox.com/debian/ceph-squid
Suites: trixie
Components: enterprise
Enabled: no
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

apt update
```

---

### Problem: `apt update` fails with GPG error

**Cause:** Missing Proxmox archive keyring

**Solutions:**
```bash
# Download and install the keyring
wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg -O /usr/share/keyrings/proxmox-archive-keyring.gpg
apt update
```

---

### Problem: NVMe not detected during Proxmox install

**Cause:** NVMe not seated properly, or M.2 slot issue

**Solutions:**
1. Power off → open HP EliteDesk → reseat NVMe in M.2 slot
2. Check BIOS → Storage → verify M.2 controller is enabled
3. Try BIOS update (HP EliteDesk 800 G4 BIOS updates available on HP support site)

---

## Next Steps

**Phase 1 complete!** Proxmox VE is running and ready for containers.

**Phase 2 — Network Services:**
- Generate SSH keys and add to Proxmox
- Download Debian 12 LXC template
- Create CT100 (network-svc, 192.168.1.100) — for AdGuard Home + Traefik
- Create CT101 (core-svc, 192.168.1.101) — for all Docker services
- Install AdGuard Home as systemd service
- Configure pfSense DNS + host overrides for `*.yourdomain.com` internal subdomains (split-DNS)
- Install Traefik via Docker

See: [phase-2-network-services.md](phase-2-network-services.md)

---

*Last updated: 2026-05-01*
*Based on: HP EliteDesk 800 G4 SFF (i5-8500, 16GB DDR4, 500GB NVMe WD Blue SN570)*
*Proxmox VE 9.1-1 on Debian 13 Trixie*
*Hostname: homelab.yourdomain.com (short: homelab)*
*Network: pfSense (192.168.1.1) → Proxmox (192.168.1.10)*
