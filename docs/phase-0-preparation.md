# Homelab Setup Guide — Phase 0
## Preparation

**Target:** Prepare hardware, download software, configure BIOS, and move domain DNS to Cloudflare
**Timeframe:** Day 1 (1–2 hours)
**Outcome:** Hardware verified, Proxmox ISO on bootable USB, domain pointing to Cloudflare DNS
**Next:** [Phase 1 — Proxmox Installation](phase-1-proxmox-installation.md)

---

## Table of Contents

1. [Domain & DNS Setup](#step-01-domain--dns-setup)
2. [BIOS Configuration](#step-02-bios-configuration)
3. [Download Proxmox VE ISO](#step-03-download-proxmox-ve-iso)
4. [Flash ISO to USB Drive](#step-04-flash-iso-to-usb-drive)
5. [Hardware Verification Checklist](#step-05-hardware-verification-checklist)

---

### Step 0.1: Domain & DNS Setup

> **Assumption:** You already own a domain from RumahWeb (you mentioned this).
> **Goal:** Point domain to Cloudflare DNS so it can manage your DNS records.

#### 0.1.1: Log into RumahWeb

1. Go to **rumahweb.com** → **Login**
2. Navigate to **Domain Management** → select your domain
3. Under **Nameservers**, note the current nameserver IPs

#### 0.1.2: Create Cloudflare Account

1. Go to **cloudflare.com** → **Sign up**
2. Enter your email + password
3. Confirm email
4. Select **Free Plan**

#### 0.1.3: Add Domain to Cloudflare

1. In Cloudflare dashboard → **Add a domain**
2. Enter your domain name (e.g., `yourdomain.com`)
3. Cloudflare will scan existing DNS records (should find 0–3 from RumahWeb)
4. Cloudflare gives you 2 new nameservers:
   ```
   ns1.cloudflare.com
   ns3.cloudflare.com
   ```

#### 0.1.4: Update Nameservers on RumahWeb

1. Return to **RumahWeb Domain Management**
2. Find **Nameserver settings**
3. Replace old nameservers with Cloudflare's:
   ```
   ns1.cloudflare.com
   ns3.cloudflare.com
   ```
4. **Save** (propagation takes 15 min–2 hours)

#### 0.1.5: Verify in Cloudflare

1. Cloudflare dashboard → your domain
2. Wait for **Status: Active** (usually ~5 min)
3. You now control DNS for your domain ✅

**Why this matters:** You'll use Cloudflare Tunnel in Phase 4 to expose dev services publicly without a real public IP.

---

### Step 0.2: BIOS Configuration

> **Goal:** Enable CPU virtualization (VT-x) so Proxmox can run virtual machines.

#### 0.2.1: Access BIOS

1. **Power off** the HP EliteDesk 800 G4 completely
2. Power on → immediately press **F10** (repeatedly, not held)
3. BIOS setup screen appears

#### 0.2.2: Navigate to Virtualization Settings

```
BIOS Menu Path:
  ► Advanced
    ► Device Options
      ► Intel Virtualization Technology → Enable
```

**What you're looking for:**
- **Intel Virtualization Technology (VT-x)** → should show **Enabled** after change
- **VT-d** → leave as **Disabled** for now (only needed later for PCIe passthrough with HDD)

#### 0.2.3: Save & Exit

1. Press **F10** (Save)
2. Confirm **Yes**
3. System reboots

**Verify after reboot:**
```bash
# After installing Linux, run this to confirm:
grep -o 'vmx' /proc/cpuinfo | head -1
# Output: vmx ← means VT-x is enabled ✅
```

---

### Step 0.3: Download Proxmox VE ISO

> **Goal:** Get the Proxmox installer. We're using Proxmox VE 9.x (latest stable).

#### 0.3.1: Download from Official Source

1. Go to **proxmox.com/en/proxmox-virtual-environment/overview**
2. Scroll to **Download** section
3. Download **Proxmox VE 9.x ISO** (latest stable)
   - File: `proxmox-ve_9.x-1.iso` (~1.2 GB)
   - Save to your laptop/desktop

**Alternative (faster):** Download via torrent from proxmox.com

#### 0.3.2: Verify ISO Integrity (Optional but Recommended)

1. In the same directory, download the **SHA256 checksum file**
2. On Linux/Mac:
   ```bash
   sha256sum -c proxmox-ve_9.x-1.iso.sha256
   ```
3. On Windows: Use **7-Zip** → right-click ISO → **CRC SHA**
4. Compare with published checksum on proxmox.com ✅

---

### Step 0.4: Flash ISO to USB Drive

> **Goal:** Create a bootable USB installer for Proxmox.
> **Hardware:** USB 3.0 drive, 4GB+ capacity

#### 0.4.1: Prepare USB Drive

1. Insert USB drive into your laptop
2. **⚠️ WARNING:** This will erase everything on the USB drive
3. Download **Balena Etcher** (free):
   - **Windows:** balena.io/etcher → download `.exe`
   - **Mac:** balena.io/etcher → download `.dmg`
   - **Linux:** `sudo apt install balena-etcher-electron`

#### 0.4.2: Flash with Etcher

1. Open **Balena Etcher**
2. Click **Select Image** → choose `proxmox-ve_9.1-1.iso`
3. Click **Select Target** → choose your USB drive (⚠️ verify drive letter!)
4. Click **Flash** → wait 3–5 minutes
5. When done → **Close**

**After flashing:**
```
USB drive now contains:
  ├── EFI partition (bootable)
  ├── ISO content (installer files)
  └── Ready to boot on HP EliteDesk
```

---

### Step 0.5: Hardware Verification Checklist

> **Before you install, confirm your HP EliteDesk is ready.**

| Check | How | Expected |
|---|---|---|
| Power supply | HP EliteDesk → power button | ✅ Boots to BIOS |
| M.2 NVMe SSD | HP EliteDesk → BIOS → Storage | ✅ Detected (show capacity) |
| RAM | HP EliteDesk → BIOS → System Info | ✅ Shows 16 GB |
| CPU | HP EliteDesk → BIOS → System Info | ✅ Intel Core i5-8500 |
| VT-x | BIOS → Advanced → Device Options | ✅ Enabled |
| USB boot order | BIOS → Boot Options | ✅ USB drive listed first |

**If NVMe not detected:**
- Check if NVMe is fully seated in M.2 slot
- Check BIOS for M.2 controller enabled
- Consult HP EliteDesk service manual if needed

---

## Phase 0 Checklist

```
[ ] Domain transferred to Cloudflare nameservers
[ ] Cloudflare status shows Active for your domain
[ ] VT-x enabled in BIOS
[ ] Proxmox VE ISO downloaded
[ ] ISO integrity verified (SHA256)
[ ] USB drive flashed with Proxmox ISO
[ ] NVMe SSD detected in BIOS
[ ] RAM shows 16 GB in BIOS
[ ] USB boot order set first
```

---

## Next Steps

**Phase 0 complete!** Your hardware is ready and USB installer is prepared.

**Phase 1 — Proxmox Installation:**
- Boot from USB and install Proxmox VE
- Configure network bridge for LXC containers
- Set static IP 192.168.1.10
- Update system and configure backup storage

See: [phase-1-proxmox-installation.md](phase-1-proxmox-installation.md)

---

*Last updated: 2026-05-01*
*Based on: HP EliteDesk 800 G4 SFF (i5-8500, 16GB DDR4, 500GB NVMe)*
*Network: IndiHome CGNAT → pfSense → LAN 192.168.1.x*
