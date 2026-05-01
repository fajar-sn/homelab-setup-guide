# Homelab Setup Guide — Phase 4
## Remote Access via Tailscale

**Target:** Install Tailscale subnet router on Proxmox host so all homelab services are reachable from anywhere
**Timeframe:** Day 3 (1–2 hours)
**Prerequisite:** [Phase 3 — Core Services](phase-3-core-services.md) complete; all services accessible on LAN at `*.yourdomain.com`
**Outcome:** Access all `*.yourdomain.com` internal services from any device, anywhere, via Tailscale — no public IP required, zero open ports
**Next:** Phase 5 — HTTPS & Let's Encrypt Certificates *(not yet written)*

---

## Table of Contents

1. [Phase 4 Overview](#phase-4-overview)
2. [Create Tailscale Account](#step-41-create-tailscale-account)
3. [Install Tailscale on Proxmox Host](#step-42-install-tailscale-on-proxmox-host)
4. [Advertise Subnet Route](#step-43-advertise-subnet-route)
5. [Approve Routes in Tailscale Admin](#step-44-approve-routes-in-tailscale-admin-console)
6. [Disable Key Expiry](#step-45-disable-key-expiry-for-subnet-router)
7. [Configure Split-DNS in Tailscale](#step-46-configure-split-dns-in-tailscale-admin)
8. [Install Tailscale on Client Devices](#step-47-install-tailscale-on-client-devices)
9. [Test Remote Access](#step-48-test-remote-access)
10. [Security Hardening](#step-49-security-hardening)
11. [Testing & Verification Checklist](#testing--verification-checklist)
12. [Troubleshooting](#troubleshooting)

---

## Phase 4 Overview

### What is Tailscale?

Tailscale creates a **private encrypted mesh network** between your devices using **WireGuard** underneath. Every device on your Tailscale network gets a `100.x.x.x` IP address. Devices can reach each other directly regardless of where they are — home, office, mobile data, hotel Wi-Fi.

A **subnet router** is a Tailscale node that shares access to a whole IP range with the rest of the Tailscale network. You'll configure the Proxmox host as a subnet router so that any device you log into Tailscale on can reach everything in your `192.168.1.0/24` LAN — as if it were physically plugged in at home.

### Why This Works Without a Public IP (CGNAT-safe)

```
Your situation:
  IndiHome → CGNAT → No real public IP
  Traditional VPN (OpenVPN, WireGuard self-hosted) → IMPOSSIBLE (can't receive inbound)

How Tailscale bypasses this:
  ┌────────────────────────────────────────────────────────────────────┐
  │  Proxmox host opens OUTBOUND connection to Tailscale relay servers │
  │  Your laptop (coffee shop) also opens OUTBOUND to same relay       │
  │  Tailscale brokers a direct peer-to-peer WireGuard tunnel          │
  │  No inbound ports opened. No public IP needed.                     │
  └────────────────────────────────────────────────────────────────────┘
```

### How Access Works After Phase 4

```
You (on phone/laptop outside home)
       │  Tailscale is ON
       ▼
Tailscale network (100.x.x.x encrypted mesh)
       │  subnet route: 192.168.1.0/24 via homelab
       ▼
Proxmox host (192.168.1.10) → routes to LAN
       │
       ▼  pfSense split-DNS: vault.yourdomain.com → 192.168.1.100
       ▼
Nginx PM (192.168.1.100) → Vaultwarden (CT101:8080)
```

The same `vault.yourdomain.com` URL that works at home works over Tailscale — no URL changes, no separate "remote" addresses.

### Access Scenarios

| Where you are | Tailscale on? | URL | Works? |
|---|---|---|---|
| Home LAN | No | `http://vault.yourdomain.com` | ✅ direct via pfSense DNS |
| Home LAN | Yes | `http://vault.yourdomain.com` | ✅ still works (Tailscale doesn't break LAN) |
| Coffee shop / mobile | Yes | `http://vault.yourdomain.com` | ✅ via Tailscale tunnel |
| Anywhere | No | `http://vault.yourdomain.com` | ❌ intentional — private only |

### Component Roles

| Component | Role |
|---|---|
| **Tailscale coordination server** | Manages device discovery and key exchange. Does NOT see your traffic content |
| **Proxmox host (`homelab`)** | Subnet router — bridges Tailscale to your `192.168.1.0/24` LAN |
| **Your laptop / phone** | Tailscale client — connects through the subnet router to reach LAN services |
| **pfSense Unbound** | DNS — resolves `*.yourdomain.com` to internal IPs, used by both LAN and Tailscale clients |

> **Privacy note:** Tailscale relay (DERP) servers are only used as fallback when direct peer-to-peer fails. When direct connection succeeds (most cases), your traffic never touches Tailscale's servers. Even when relayed, all traffic is end-to-end WireGuard encrypted — Tailscale cannot read it.

---

## Step 4.1: Create Tailscale Account

### 4.1.1: Sign Up

1. Open browser → **https://tailscale.com**
2. Click **Get started**
3. Sign in with **Google**, **GitHub**, or **Microsoft** account
   - Use your personal account — the one you'll use on all your devices
4. Choose plan: **Free** (up to 100 devices, more than enough for a personal homelab)
5. Complete sign-up

> **Important:** Every device you add (Proxmox host, laptop, phone) must sign in with **the same Tailscale account** to be on the same private network.

### 4.1.2: Tailscale Admin Console

After sign-up you land on: **https://login.tailscale.com/admin/machines**

This is your control panel. You will come back here to:
- See all connected devices
- Approve subnet routes
- Configure DNS

Keep this tab open — you'll use it throughout this phase.

---

## Step 4.2: Install Tailscale on Proxmox Host

The Proxmox host (`homelab`, `192.168.1.10`) will be the **subnet router**. Install Tailscale directly on the Proxmox host — it already has full kernel access and bridges to all containers.

### 4.2.1: SSH into Proxmox

From your laptop (on home LAN):

```bash
ssh root@192.168.1.10
```

You should see:
```
root@homelab:~#
```

### 4.2.2: Verify IP Forwarding is Enabled

IP forwarding allows the Proxmox host to pass packets between Tailscale and your LAN. Proxmox normally enables this automatically for LXC/VM networking. Verify:

```bash
sysctl net.ipv4.ip_forward
```

Expected output:
```
net.ipv4.ip_forward = 1
```

If it shows `0`, enable it now:

```bash
echo 'net.ipv4.ip_forward = 1' | tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | tee -a /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### 4.2.3: Install Tailscale

The official install script auto-detects your Debian version and installs the latest stable release:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

This will:
1. Add Tailscale's apt repository for Debian (auto-detects Trixie)
2. Install the `tailscale` package (latest stable)
3. Start and enable the `tailscaled` system service

Expected output (last few lines):
```
Installation complete! Log in to start using Tailscale by running:

sudo tailscale up
```

### 4.2.4: Verify Installation

```bash
tailscale version
```

Example output (version number will vary):
```
1.82.0
  tailscale commit: abc1234...
  go version: go1.22.x
```

```bash
systemctl status tailscaled
```

Expected: `Active: active (running)`

---

## Step 4.3: Advertise Subnet Route

This single command connects Proxmox to Tailscale AND declares it as a subnet router for your LAN.

### 4.3.1: Start Tailscale with Subnet Advertisement

```bash
tailscale up \
  --advertise-routes=192.168.1.0/24 \
  --accept-dns=false
```

**What each flag does:**

| Flag | Purpose |
|---|---|
| `--advertise-routes=192.168.1.0/24` | Tell Tailscale: "I can route traffic destined for this IP range" — your whole LAN |
| `--accept-dns=false` | Do NOT let Tailscale override this machine's DNS. Proxmox should keep using pfSense as its DNS, not Tailscale |

### 4.3.2: Authenticate

The command outputs an authentication URL:
```
To authenticate, visit:

	https://login.tailscale.com/a/abc123xyz...
```

1. Copy the URL
2. Open it in your browser (on your laptop — you don't need to be on any specific device)
3. You're already logged in from Step 4.1 — click **Connect**
4. Return to the Proxmox terminal — it shows:
   ```
   Success.
   ```

### 4.3.3: Confirm Tailscale is Connected

```bash
tailscale status
```

Expected output:
```
100.x.x.x   homelab               yourname@   linux   -
```

Write down your Proxmox Tailscale IP (the `100.x.x.x` address):

```
Proxmox Tailscale IP: 100.___.___.___ 
```

You can SSH to this IP from anywhere on Tailscale as an alternative to `192.168.1.10`.

---

## Step 4.4: Approve Routes in Tailscale Admin Console

Routes are not automatically active — Tailscale requires you to manually approve them in the admin console. This is a deliberate security gate (prevents a compromised device from advertising arbitrary routes).

### 4.4.1: Open Admin Console

Browser → **https://login.tailscale.com/admin/machines**

You should see `homelab` listed with a green **Connected** status.

### 4.4.2: Open Route Settings

1. Find `homelab` in the machine list
2. Click the **`...`** (three dots) menu on the right side of the row
3. Click **Edit route settings...**

A panel appears:
```
Subnet routes

  ○ 192.168.1.0/24                               [toggle]
```

4. Click the toggle next to `192.168.1.0/24` to **enable** it (it turns blue/on)
5. Click **Save**

### 4.4.3: Verify Route Approved

Back in the machines list, `homelab` should now display:
```
Subnets: 192.168.1.0/24
```

---

## Step 4.5: Disable Key Expiry for Subnet Router

Tailscale device keys expire every **180 days** by default. When a key expires, that device is disconnected from Tailscale until someone manually re-authenticates it. For user laptops and phones this is fine as a security feature. For an always-on server, it means losing remote access if you're away when it expires.

### 4.5.1: Disable Key Expiry

1. Admin console → **https://login.tailscale.com/admin/machines**
2. Find `homelab` → click **`...`** → **Disable key expiry**
3. Confirm the dialog

`homelab` now shows a **No expiry** badge in the machine list.

> **Why this matters:** If you're traveling and the key expires, you have no way to get back in remotely. The Proxmox host would need a physical keyboard/monitor to re-authenticate.

---

## Step 4.6: Configure Split-DNS in Tailscale Admin

This step makes `*.yourdomain.com` resolve correctly on your Tailscale clients when you're away from home. Without it, your laptop at a coffee shop would ask Cloudflare's public DNS for `vault.yourdomain.com`, get no answer (there's no public A record), and fail.

With this step, Tailscale tells your device: "For anything ending in `.yourdomain.com`, ask `192.168.1.1` (pfSense) instead." pfSense returns the internal IP, and traffic routes through the Tailscale subnet tunnel.

### 4.6.1: Open DNS Settings

Browser → **https://login.tailscale.com/admin/dns**

### 4.6.2: Add Custom Nameserver

Under the **Nameservers** section, click **Add nameserver** → **Custom...**

Fill in:

| Field | Value |
|---|---|
| **Nameserver IP** | `192.168.1.1` |
| **Restrict to domain** | `yourdomain.com` |

Click **Save**

> The "Restrict to domain" field is what makes this **split-DNS** — only queries for `*.yourdomain.com` go to pfSense. Everything else (google.com, etc.) goes through normal internet DNS. Your Tailscale clients don't lose normal browsing when connected.

### 4.6.3: Verify DNS Configuration

The DNS page should now show under Nameservers:
```
192.168.1.1    yourdomain.com
```

**How the full DNS path works for Tailscale clients:**
```
Client queries: vault.yourdomain.com
    │
    ▼  Tailscale sees: "yourdomain.com → ask 192.168.1.1"
    │
    ▼  Query routed to 192.168.1.1 via subnet route (192.168.1.0/24)
    │
    ▼  pfSense Unbound host override: vault.yourdomain.com → 192.168.1.100
    │
    ▼  Client connects to 192.168.1.100 (Nginx PM)
    │
    ▼  Nginx PM → CT101:8080 (Vaultwarden)
```

> **Do NOT enable MagicDNS** for your domain. MagicDNS is Tailscale's own `*.ts.net` hostname system and is separate. Your `yourdomain.com` split-DNS configuration does not conflict with it.

---

## Step 4.7: Install Tailscale on Client Devices

Install Tailscale on every device you want to use for remote access. Sign in with the **same account** on every device.

### 4.7.1: Windows Laptop

1. Go to **https://tailscale.com/download/windows**
2. Download and run the installer (`.exe`)
3. Follow the installation prompts
4. After install, a Tailscale icon appears in the **system tray** (bottom-right near the clock)
5. Click the tray icon → **Log in**
6. Sign in with the same account as Step 4.1
7. Tailscale connects automatically

**Enable subnet access on Windows:**

After connecting, right-click the Tailscale tray icon:
- Look for **Use Tailscale subnets** or **Accept routes** — make sure it is **checked/enabled**
- If you don't see it: click the tray icon → click **`...`** (three dots) → tick **Use subnet routes**

Without this, Windows won't route `192.168.1.x` traffic through Tailscale.

### 4.7.2: iPhone

1. App Store → search **Tailscale** (developer: Tailscale Inc.)
2. Install → Open
3. Tap **Get started** → sign in with same account
4. iOS will ask permission to add a VPN configuration → tap **Allow**
5. Toggle Tailscale **ON**

### 4.7.3: Android

1. Play Store → search **Tailscale** (developer: Tailscale Inc.)
2. Install → Open → sign in with same account
3. Android asks for VPN permission → tap **OK**
4. Toggle Tailscale **ON**

> **Mobile note:** Tailscale uses the device's VPN slot. Only one VPN can be active at a time. When Tailscale is on, your homelab is reachable. Regular internet browsing still works normally through Tailscale (it's not a full tunnel by default — only LAN traffic goes through the subnet).

---

## Step 4.8: Test Remote Access

> **Important:** This test must be done from outside your home LAN to be meaningful.
>
> - Turn off home Wi-Fi on your phone → use mobile data only, then test from your phone
> - Or: use a laptop at a coffee shop / different building
> - Or at home: connect your laptop to a phone hotspot (disconnected from home Wi-Fi)

### 4.8.1: Verify Tailscale is Connected on Client

**Windows:** The Tailscale tray icon should be filled/dark (not greyed out).
Click the icon — it shows:
```
Connected
This device: 100.x.x.x
```

**Phone:** Tailscale app shows a green/filled icon or "Connected" status.

### 4.8.2: Test IP Routing First (Before DNS)

Before testing domain names, confirm the subnet tunnel works at the IP level:

**Windows (PowerShell or cmd):**
```powershell
ping 192.168.1.10
```
Expected: replies from `192.168.1.10` (your Proxmox host)

```powershell
ping 192.168.1.100
```
Expected: replies from `192.168.1.100` (CT100, Nginx PM)

If pings fail → go to Troubleshooting. Don't proceed until pings work.

### 4.8.3: Test DNS Resolution

```powershell
nslookup vault.yourdomain.com
```

Expected output:
```
Server:  100.100.100.100
Address:  100.100.100.100#53

Non-authoritative answer:
Name:    vault.yourdomain.com
Address:  192.168.1.100
```

> `100.100.100.100` is Tailscale's internal DNS resolver. It forwards your `yourdomain.com` query to pfSense (`192.168.1.1`) and returns the result.

If DNS returns `192.168.1.100` → split-DNS is working. ✅

### 4.8.4: Access All Services in Browser

| URL | Expected result |
|---|---|
| `http://vault.yourdomain.com` | Vaultwarden login page |
| `http://immich.yourdomain.com` | Immich web UI |
| `http://pdf.yourdomain.com` | Stirling-PDF dashboard |
| `http://status.yourdomain.com` | Uptime Kuma dashboard |
| `http://portainer.yourdomain.com` | Portainer login |
| `https://192.168.1.10:8006` | Proxmox web dashboard |

### 4.8.5: Test SSH over Tailscale

```powershell
ssh root@192.168.1.10
```

Or using the Tailscale IP directly (doesn't need pfSense DNS):
```powershell
ssh root@100.x.x.x   # your Proxmox Tailscale IP from Step 4.3.3
```

Both should work. ✅

---

## Step 4.9: Security Hardening

### 4.9.1: Understand What is Exposed

Nothing is publicly accessible. Verify:

| Attack surface | Status |
|---|---|
| Open inbound ports on pfSense | ❌ None (no port forwarding added) |
| Public DNS records for `*.yourdomain.com` | ❌ None (Cloudflare has no A records for internal services) |
| Tailscale access | ✅ Requires account login on an approved device |
| Tailscale relay traffic | ✅ End-to-end WireGuard encrypted; Tailscale servers cannot decrypt it |

### 4.9.2: Verify Tailscale Starts on Boot

```bash
# On Proxmox host
systemctl is-enabled tailscaled
```

Expected: `enabled`

If `disabled`:
```bash
systemctl enable tailscaled
```

### 4.9.3: Access Control Lists (ACLs) — Optional

By default, all devices on your Tailscale account can reach all other devices. For a personal setup with only your own devices this is fine.

If you ever add devices for other people (family, colleagues):
1. Admin console → **https://login.tailscale.com/admin/acls**
2. The default policy (`"action": "accept"` for all traffic between your own devices) is appropriate for now
3. If you add other users, restrict subnet access using Tailscale's ACL docs: **https://tailscale.com/kb/1018/acls**

### 4.9.4: Tailscale Free Tier Limits

| Limit | Free tier |
|---|---|
| Devices | 100 |
| Users | 1 (personal account) |
| Subnet routers | 2 |
| Relay bandwidth | Unlimited |

A personal homelab will stay well within free tier indefinitely.

---

## Testing & Verification Checklist

Perform all tests from **outside your home network** (phone on mobile data, or laptop on a different Wi-Fi):

```
[ ] tailscale status on Proxmox shows: homelab  connected
[ ] homelab shows "No expiry" badge in Tailscale admin console
[ ] 192.168.1.0/24 subnet shows as approved in Tailscale admin → homelab → route settings
[ ] Custom nameserver 192.168.1.1 for yourdomain.com saved in Tailscale admin DNS

[ ] ping 192.168.1.10 succeeds from Tailscale client (subnet routing works)
[ ] ping 192.168.1.100 succeeds from Tailscale client
[ ] nslookup vault.yourdomain.com returns 192.168.1.100 (split-DNS works)

[ ] http://vault.yourdomain.com → Vaultwarden login ✅
[ ] http://immich.yourdomain.com → Immich web UI ✅
[ ] http://pdf.yourdomain.com → Stirling-PDF ✅
[ ] http://status.yourdomain.com → Uptime Kuma ✅
[ ] https://192.168.1.10:8006 → Proxmox dashboard ✅
[ ] SSH root@192.168.1.10 works over Tailscale ✅

[ ] Tailscale installed on laptop ✅
[ ] Tailscale installed on phone ✅
[ ] "Use subnet routes" / "Accept routes" enabled on Windows client ✅
```

---

## Troubleshooting

### Problem: `ping 192.168.1.10` fails from Tailscale client

The subnet tunnel isn't routing.

**Check 1 — Subnet approved in admin console?**
- Admin console → **https://login.tailscale.com/admin/machines**
- Click `...` next to `homelab` → **Edit route settings**
- `192.168.1.0/24` must be toggled **ON** (blue)

**Check 2 — "Accept routes" enabled on the client?**
- Windows: right-click Tailscale tray icon → confirm **Use subnet routes** is checked
- If option not visible: click tray icon → `...` → **Use subnet routes**
- Mac: Tailscale menu bar icon → **Preferences** → **Use Tailscale subnets** ✓
- Mobile: Tailscale app → Settings → Routes — accept all advertised routes

**Check 3 — Tailscale running on Proxmox?**
```bash
# On Proxmox host
tailscale status
systemctl status tailscaled
```
If `tailscaled` is stopped: `systemctl start tailscaled` then `tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false`

---

### Problem: `nslookup vault.yourdomain.com` returns no answer or NXDOMAIN

Split-DNS isn't routing queries to pfSense.

**Check 1 — Nameserver configured correctly?**
- Admin console → **https://login.tailscale.com/admin/dns**
- Under Nameservers: should show `192.168.1.1` with domain restriction `yourdomain.com`
- If missing: re-add using Step 4.6.2

**Check 2 — pfSense reachable via Tailscale?**
```powershell
ping 192.168.1.1
```
If this fails, the IP routing issue (above) must be fixed first — DNS over Tailscale can only work if `192.168.1.1` is reachable.

**Check 3 — Test pfSense DNS directly:**
```powershell
nslookup vault.yourdomain.com 192.168.1.1
```
If this returns `192.168.1.100` but Tailscale DNS doesn't → the split-DNS config in Tailscale admin is wrong.

**Check 4 — Reconnect Tailscale on client:**
Disconnect and reconnect Tailscale on your client device — DNS config changes sometimes need a reconnect to take effect.

---

### Problem: Services load at 192.168.x.x IPs but not at yourdomain.com URLs

Subnet routing works, but split-DNS isn't forwarding to pfSense. Confirm:

1. Tailscale admin DNS page → nameserver `192.168.1.1` with domain restriction `yourdomain.com` is present
2. pfSense → Services → DNS Resolver → Host Overrides — all entries use Domain = `yourdomain.com` (not `lan`)
3. Disconnect/reconnect Tailscale on client to reload DNS settings

---

### Problem: Key expiry warning or Tailscale disconnected after 180 days

If you skipped Step 4.5 or the key already expired:

1. Physical access to Proxmox (or from LAN): `tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false`
2. Admin console → disable key expiry as per Step 4.5
3. Confirm `tailscale status` shows connected

---

### Problem: After Proxmox reboot, remote access broken

`tailscaled` should restart automatically. Verify:
```bash
systemctl status tailscaled
tailscale status
```

If `tailscale status` shows the machine connected but subnet isn't working:
```bash
tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false
```

If `tailscaled` is not running:
```bash
systemctl start tailscaled
systemctl enable tailscaled
```

---

*Hostname: homelab.yourdomain.com (short: homelab)*
*LAN: 192.168.1.0/24 | Subnet router: 192.168.1.10 (Proxmox host)*
*Tailscale: free personal tier — https://login.tailscale.com/admin*
