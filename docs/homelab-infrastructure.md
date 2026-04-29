# Homelab Infrastructure

Self-hosted infrastructure for the questlab.local environment running on server-quest (Proxmox). Covers virtualization, container services, networking, storage, and the runbooks needed to deploy, maintain, and recover each system. Owner: QuestOnThis.

# Hosts & Hypervisors

# Proxmox Storage Setup — pve

**Status:** ✅ Running
**Owner:** Eric Del Angel
**Host:** `pve` (Proxmox VE 9.1.7)
**Drive:** Seagate Barracuda Green 2TB HDD (`/dev/sda`, serial `5YD20DC1`)
**Date:** 2026-04-15

---

## Overview

Initial Proxmox storage build on the 2TB Seagate. The drive came in
with a leftover LVM volume group from a previous Proxmox install that
was holding partition `/dev/sda3` and blocking reuse. Cleaned it out,
wiped the disk, and rebuilt as a single-disk ZFS pool named `tank`
with separate datasets for ISO images, backups, templates, and snippets.

---

## Problem

Proxmox UI showed:

```
disk/partition '/dev/sda3' has a holder (500)
```

Cause: leftover LVM volume group `pve-OLD-5858D26D` from a previous
Proxmox install was still claiming the partition.

---

## Cleanup Steps

```bash
# Identify the holder
ls -l /sys/block/sda/sda3/holders/      # showed dm-4 through dm-7
pvs                                      # confirmed pve-OLD-5858D26D on sda3

# Remove old LVM
vgchange -an pve-OLD-5858D26D
vgremove -f pve-OLD-5858D26D
pvremove -f /dev/sda3
wipefs -a /dev/sda3

# Wipe the entire disk
sgdisk --zap-all /dev/sda
```

---

## ZFS Pool Creation

Built via Proxmox UI: **Datacenter → pve → Disks → ZFS → Create**

| Setting | Value |
|---|---|
| Name | `tank` |
| RAID Level | Single Disk |
| Compression | on (lz4) |
| ashift | 12 |
| Add Storage | ✅ |

---

## Tuning

```bash
zfs set atime=off tank
echo "options zfs zfs_arc_max=2147483648" > /etc/modprobe.d/zfs.conf
update-initramfs -u
# ARC cap (2GB) takes effect after next reboot
```

---

## File-Level Datasets

ZFS pool storage in Proxmox only supports `Disk image` + `Container`
content types. Created separate datasets for file-based content:

```bash
zfs create tank/iso
zfs create tank/backups
zfs create tank/templates
zfs create tank/snippets
```

---

## Final Storage Layout

| Storage ID | Type | Path | Content |
|---|---|---|---|
| `tank` | ZFS | — | Disk image, Container |
| `tank-iso` | Directory | `/tank/iso` | ISO image |
| `tank-backup` | Directory | `/tank/backups` | VZDump backup file |
| `tank-templates` | Directory | `/tank/templates` | Container template |
| `tank-snippets` | Directory | `/tank/snippets` | Snippets |

---

## Drive Health Note

⚠️ Drive is aged: 56,438 power-on hours (~6.4 years), 760 reallocated
sectors. Treat as **lab/scratch storage only**. Not suitable for
irreplaceable data without external backups.

---

## VM Inventory

| VMID | Name | Role |
|---|---|---|
| 100 | Win11PRO | Windows 11 Pro client |
| 101 | ubuntu-server | Ubuntu Server |
| 102 | ubuntu-desktop | Ubuntu Desktop |
| 103 | windows-11 / WIN11PRO-2 | Second Windows 11 client |
| 104 | WIN2022 | Domain controller (`PCLAB`) |

---

## Useful Commands

```bash
zpool status tank
zpool list
zfs list
zfs get compression,atime tank
zpool scrub tank
smartctl -a /dev/sda
qm list
qm config 
```

📁 Chapter: Hosts & Hypervisors
Page name: ISO Files — Lab VMs
markdown# ISO Files — Lab VMs

**Status:** ✅ Available

# Hosts & Hypervisors

**Status:** ✅ Available
**Host:** `pve`
**Storage:** `tank-iso` (`/tank/iso`)
**Last Updated:** 2026-04-15

---

## Overview

ISO images downloaded for the lab VM lineup: Windows desktop, Windows
Server, Ubuntu desktop, Ubuntu Server, plus the VirtIO driver ISO
required for any Windows VM running on Proxmox.

All ISOs live under the `tank-iso` storage on the `tank` ZFS pool.

---

## Download Method

In Proxmox UI: sidebar → **`tank-iso (pve)`** → **ISO Images** tab →
**Download from URL** → paste URL → **Query URL** → **Download**.

For Windows 11 (no direct URL allowed by Microsoft), download the ISO
manually on a workstation, then upload via the same panel using the
**Upload** button.

---

## ISO Inventory

| OS | URL / Source | Notes |
|---|---|---|
| Ubuntu Desktop 24.04 | `https://releases.ubuntu.com/24.04/ubuntu-24.04.3-desktop-amd64.iso` | Direct download |
| Ubuntu Server 24.04 | `https://releases.ubuntu.com/24.04/ubuntu-24.04.3-live-server-amd64.iso` | Direct download |
| Windows Server 2025 (eval) | `https://go.microsoft.com/fwlink/?linkid=2293312` | 180-day evaluation |
| Windows Server 2022 (eval) | https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022 | 180-day eval, used for VM 104 (`PCLAB`) |
| Windows 11 (multi-edition) | https://www.microsoft.com/software-download/windows11 | Manual download, ~7-8 GB |
| VirtIO drivers | `https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso` | Required for Windows VMs on Proxmox |

---

## VirtIO Driver Notes

Windows installers do not include drivers for the high-performance
VirtIO devices Proxmox exposes (disk, network, balloon, etc.). The
`virtio-win.iso` is mounted as a **second CD drive** during Windows
installation so the installer can load disk drivers and continue.

Without this ISO, the Windows installer reports **"No drives found"**
when partitioning.

---

## Windows 11 Caveats

- Windows 11 requires **TPM 2.0** and **Secure Boot**. In Proxmox VM
  hardware: BIOS = `OVMF (UEFI)`, EFI Disk added, TPM v2.0 added,
  Machine = `q35`.
- The downloaded ISO is multi-edition; it contains Home, Pro,
  Education, Enterprise. **Pick Pro during install.** If the installer
  skips the edition picker, use the Microsoft generic install key
  `VK7JG-NPHTM-C97JM-9MPGT-3V66T` to direct it to Pro (does not activate).
- Win11 **Home cannot join AD domains** — Pro is required.

---

## Win11 OOBE Local-Account Bypass

To skip Microsoft account requirement during install:

1. At the "Let's connect you to a network" screen, press `Shift + F10`
2. Type: `start ms-cxh:localonly` → Enter
3. A "Create a local account for this PC" dialog appears

Older `oobe\bypassnro` method removed in newer 24H2 builds.

# Server Build

# Domain Controller — PCLAB Build

**Status:** ✅ Running
**VM:** 104 (`WIN2022`)
**Hostname:** `PCLAB`
**FQDN:** `PCLAB.homelab.local`
**Domain:** `homelab.local`
**Static IP:** `192.168.1.2/24`
**Gateway:** `192.168.1.1`
**Roles:** AD DS, DNS, DHCP
**Last Updated:** 2026-04-20

---

## Overview

First domain controller for the `homelab.local` forest. Hosts AD DS,
integrated DNS with public forwarders, and DHCP for the LAN. Built on
Proxmox VM 104 with disks on the `tank` ZFS pool.

---

## VM Specs

| Resource | Value |
|---|---|
| VMID | 104 |
| Name | WIN2022 |
| RAM | 8 GB (8032 MB) |
| Boot disk | 250 GB on `tank` |
| OS | Windows Server 2022 Standard (eval) |

---

## Pre-flight: Static IP

Settings → Network & Internet → Ethernet → Edit IP settings.

| Setting | Value |
|---|---|
| IP assignment | Manual |
| IP address | `192.168.1.2` |
| Subnet prefix length | `24` |
| Gateway | `192.168.1.1` |
| Preferred DNS | `127.0.0.1` |
| Alternate DNS | `1.1.1.1` |

The `127.0.0.1` primary DNS is intentional. A DC must point to itself
for DNS or AD replication, dynamic DNS registration, and GPO will fail.

---

## Rename to PCLAB

Done before AD promotion — renaming after promotion embeds the wrong
hostname in AD/SYSVOL paths.

```powershell
Rename-Computer -NewName "PCLAB" -Restart
hostname     # verify
```

---

## Enable RDP for Remote Admin

Required for paste-friendly admin from the host workstation.

```powershell
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" `
  -Name fDenyTSConnections -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Get-NetFirewallRule -DisplayGroup "Remote Desktop" | Select DisplayName, Enabled
```

Connect from workstation: `mstsc` → `192.168.1.2` → `homelab\Administrator`.

---

## Snapshot Convention

Take Proxmox snapshots before risky changes. **Include RAM** for
"resume to exact moment" rollback.

```bash
# CLI
qm snapshot 104 DC-DNS-Working-2026-04-20 \
  --description "DC has working DNS forwarders, pre-DHCP install" \
  --vmstate 1

qm listsnapshot 104
qm rollback 104 DC-DNS-Working-2026-04-20
qm delsnapshot 104 OLD-NAME
```

GUI: VM 104 → **Snapshots** → **Take Snapshot** → ✅ Include RAM.

Naming: `{role}-{state}-{YYYY-MM-DD}` (e.g., `DC-Pre-DHCP-Install-2026-04-20`).

# AD & Identity

# AD DS Promotion & DNS Configuration

**Status:** ✅ Running
**Domain:** `homelab.local`
**Forest functional level:** Windows Server 2016+
**DC:** PCLAB (`192.168.1.2`)
**DNS forwarders:** `1.1.1.1`, `8.8.8.8`
**Last Updated:** 2026-04-20

---

## Overview

PCLAB promoted as the first DC in a new `homelab.local` forest. DNS
Server role installed automatically with AD DS and is AD-integrated.

---

## Install AD DS Role

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

---

## Promote to Domain Controller

```powershell
Install-ADDSForest `
  -DomainName "homelab.local" `
  -DomainNetbiosName "HOMELAB" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -CreateDnsDelegation:$false `
  -DatabasePath "C:\Windows\NTDS" `
  -LogPath "C:\Windows\NTDS" `
  -SysvolPath "C:\Windows\SYSVOL" `
  -Force:$true
```

Set a strong **DSRM password** when prompted. Required to boot the DC
into recovery mode. Store in password manager.

Server reboots automatically when promotion finishes.

---

## Verify After Reboot

```powershell
Get-ADDomain
Get-ADForest
nslookup homelab.local
nslookup _ldap._tcp.dc._msdcs.homelab.local
```

---

## DNS Forwarder Issue Encountered

After promotion, internal DNS resolved but external lookups timed out.

### Symptoms

- `ping 8.8.8.8` → ✅ works (routing OK)
- `nslookup google.com` → ❌ times out (server `192.168.1.2` unreachable)
- `nslookup google.com 8.8.8.8` → ✅ works (bypassing local DNS)

### Root cause

DNS Server role had no forwarders configured. Without forwarders, the
DC tried iterative resolution via root hints, which failed/timed out.

### Fix

```powershell
Set-DnsServerForwarder -IPAddress 1.1.1.1, 8.8.8.8
Get-DnsServerForwarder

Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
  -ServerAddresses ("127.0.0.1", "1.1.1.1")

Clear-DnsServerCache -Force
ipconfig /flushdns

Resolve-DnsName google.com
ping google.com
nslookup homelab.local
```

After fix: snapshot taken as `DC-DNS-Working-2026-04-20` before
proceeding to DHCP install.

---

## Critical DNS Rules for DCs

1. DC's NIC primary DNS must be `127.0.0.1` (loopback) or its own IP —
   never the router, never `8.8.8.8`.
2. Forwarders configured on the DNS service for everything outside
   `homelab.local`.
3. AD-integrated zones replicate automatically with AD.

# OU Structure & Tiered Admin Model

**Status:** ✅ Built
**Domain:** `homelab.local`
**Top-level branch:** `PUBMIX` + `_Admin` + `Disabled`
**Last Updated:** 2026-04-20

---

## Overview

Two-pattern OU design:

- **`_Admin`** — Microsoft tiered admin model (T0 / T1 / T2). Cuts
  across geography. Holds privileged accounts and the systems they
  manage.
- **`PUBMIX`** — geographic hierarchy holding regular users,
  workstations, servers, service accounts, and department groups.
- **`Disabled`** — staging OU for decommissioned objects, with
  blocked GPO inheritance.

---

## Full Tree

```
homelab.local
│
├── _Admin
│   ├── Tier0-Identity
│   │   ├── Accounts
│   │   ├── Groups
│   │   └── Servers
│   ├── Tier1-Servers
│   │   ├── Accounts
│   │   ├── Groups
│   │   └── Servers
│   └── Tier2-Workstations
│       ├── Accounts
│       ├── Groups
│       └── Workstations
│
├── PUBMIX
│   ├── USA
│   │   └── CA
│   │       └── Ventura
│   │           ├── Users
│   │           ├── Workstations
│   │           ├── Servers
│   │           ├── ServiceAccounts
│   │           └── Groups
│   │               ├── Security
│   │               └── Departments
│   │
│   └── CHILE
│       └── Valparaiso-Region
│           └── Vina-del-Mar
│               ├── Users
│               ├── Workstations
│               ├── Servers
│               ├── ServiceAccounts
│               └── Groups
│                   ├── Security
│                   └── Departments
│
└── Disabled
    ├── Users
    └── Computers
```

---

## Tier Groups

All **Global / Security**. Located in `_Admin/TierX/Groups`.

**Tier 0 — Identity:**
- `T0-Domain-Admins`
- `T0-Enterprise-Admins`
- `T0-PKI-Admins`
- `T0-Identity-Admins`

**Tier 1 — Servers:**
- `T1-Server-Admins`
- `T1-SQL-Admins`
- `T1-Backup-Operators`

**Tier 2 — Workstations:**
- `T2-Workstation-Admins`
- `T2-Helpdesk`

---

## Department Groups

Created as **security groups**, not OUs. Located in
`PUBMIX/.../{City}/Groups/Departments`.

Suffix groups with city name to avoid collisions across regions.

**Ventura:**
- `Accounting-Ventura`, `C-Level-Ventura`, `Engineer-Ventura`,
  `Finance-Ventura`, `HR-Ventura`, `Legal-Ventura`,
  `Management-Ventura`, `Marketing-Ventura`, `Sales-Ventura`

**Vina del Mar:**
- Same set with `-Vina-del-Mar` suffix.

---

## Account Naming Conventions

| Suffix | Purpose | Example | Location |
|---|---|---|---|
| `-U` | Regular daily-driver | `Eric Del Angel-U` | `PUBMIX/USA/CA/Ventura/Users` |
| `-PA` | Privileged Admin | `Mauricio Undurraga-PA` | `_Admin/TierX/Accounts` |
| `-t0` | Tier 0 admin | `edelangel-t0` | `_Admin/Tier0-Identity/Accounts` |
| `-t1` | Tier 1 admin | `edelangel-t1` | `_Admin/Tier1-Servers/Accounts` |
| `-t2` | Tier 2 admin | `edelangel-t2` | `_Admin/Tier2-Workstations/Accounts` |

The regular account has zero admin rights. RunAs the tiered account
when admin tasks are needed.

---

## Default Container Redirection

Run once on the DC:

```powershell
redirusr "OU=Users,OU=Ventura,OU=CA,OU=USA,OU=PUBMIX,DC=homelab,DC=local"
redircmp "OU=Workstations,OU=Ventura,OU=CA,OU=USA,OU=PUBMIX,DC=homelab,DC=local"
```

Auto-created users/computers now land in the Ventura branch instead
of the unmanageable default `Users` / `Computers` containers.

---

## Disabled OU

Block GPO inheritance:

1. GPMC → right-click `Disabled` OU
2. Select **Block Inheritance**

Decommission workflow: disable account → move to `Disabled/Users` →
wait 30–90 days → delete.

---

## OU vs Group — Quick Visual Tells (ADUC)

- **OU icon** — yellow folder with a small book/page graphic
- **Group icon** — yellow folder with two-people silhouettes
- **New Object dialog title bar** — explicitly says
  "Organizational Unit" or "Group"

If an OU was created when a Group was intended:

1. View → **Advanced Features** (toggle ON)
2. Right-click object → Properties → **Object** tab
3. Uncheck **Protect object from accidental deletion** → Apply
4. Right-click → Delete
5. Recreate with the correct type

# Networking Services

# DHCP Migration — Router to PCLAB

**Status:** ✅ Running
**DHCP server:** PCLAB (`192.168.1.2`)
**Scope:** `192.168.1.100 – 192.168.1.200`
**Authorized in AD:** ✅
**Last Updated:** 2026-04-20

---

## Overview

DHCP authority migrated from the consumer router to PCLAB's Windows
DHCP role. Router DHCP disabled on LAN, guest, and IoT networks.
Router operates as gateway/NAT only.

---

## Cascade Failure Encountered (Important)

Initial migration attempt caused full LAN outage. Root cause: router
DHCP disabled before DC's DHCP role was installed and authorized,
leaving nothing to lease IPs. DC's static IP also reverted to DHCP
during a network event, causing the DC itself to lose addressing.

### Recovery sequence

1. Re-enabled router DHCP temporarily to restore LAN.
2. Reapplied static IP on DC via PowerShell (more durable than GUI).
3. Confirmed DC could `ping google.com` (DNS forwarder fix —
   see AD DS page).
4. Then proceeded with DHCP role install while router DHCP still
   running as fallback.

**Lesson:** never disable a working DHCP server until the replacement
is fully tested and leasing.

---

## Static IP — Durable Apply

```powershell
$Adapter = "Ethernet"
$IP = "192.168.1.2"
$Prefix = 24
$Gateway = "192.168.1.1"

Get-NetIPAddress -InterfaceAlias $Adapter -AddressFamily IPv4 -ErrorAction SilentlyContinue |
  Remove-NetIPAddress -Confirm:$false
Get-NetRoute -InterfaceAlias $Adapter -AddressFamily IPv4 -ErrorAction SilentlyContinue |
  Where-Object DestinationPrefix -eq "0.0.0.0/0" | Remove-NetRoute -Confirm:$false

New-NetIPAddress -InterfaceAlias $Adapter -IPAddress $IP `
  -PrefixLength $Prefix -DefaultGateway $Gateway

Set-DnsClientServerAddress -InterfaceAlias $Adapter `
  -ServerAddresses ("127.0.0.1", "1.1.1.1")
```

Bulletproof against `Network Reset`:

```powershell
$AdapterGuid = (Get-NetAdapter -Name $Adapter).InterfaceGuid
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\$AdapterGuid" `
  -Name "EnableDHCP" -Value 0
```

---

## Install + Authorize DHCP Role

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools

Add-DhcpServerInDC -DnsName "PCLAB.homelab.local" `
  -IPAddress 192.168.1.2

# Clear post-install nag
Set-ItemProperty -Path "HKLM:\Software\Microsoft\ServerManager\Roles\12" `
  -Name ConfigurationState -Value 2

Get-DhcpServerInDC          # confirm authorization
```

---

## Create Scope

```powershell
Add-DhcpServerv4Scope `
  -Name "Homelab-Main" `
  -Description "Primary LAN scope" `
  -StartRange 192.168.1.100 `
  -EndRange 192.168.1.200 `
  -SubnetMask 255.255.255.0 `
  -State Active `
  -LeaseDuration 8.00:00:00
```

---

## Set Scope Options

```powershell
# Server-level (DNS + search domain)
Set-DhcpServerv4OptionValue `
  -DnsServer 192.168.1.2 `
  -DnsDomain "homelab.local"

# Scope-level (gateway)
Set-DhcpServerv4OptionValue -ScopeId 192.168.1.0 -Router 192.168.1.1

Restart-Service DHCPServer
```

---

## IP Plan

| Range | Purpose |
|---|---|
| `.1` | Router |
| `.2` | PCLAB (DC) — static |
| `.10–.49` | Static — future servers, NAS |
| `.50–.99` | Static — infrastructure (printers, switches, APs) |
| `.100–.200` | DHCP scope (workstations, VMs, phones) |
| `.201–.254` | Reserved |

---

## Verify

```powershell
Get-DhcpServerv4Scope
Get-DhcpServerv4OptionValue -ScopeId 192.168.1.0
Get-DhcpServerv4Lease -ScopeId 192.168.1.0   # see active leases
```

From a client:

```cmd
ipconfig /release
ipconfig /renew
ipconfig /all
```

`DHCP Server` should show `192.168.1.2`. `DNS Servers` should show
`192.168.1.2`. `Domain` should show `homelab.local`.

---

## Disable Router DHCP — Don't Forget Hidden Toggles

In router admin:

- **LAN → DHCP Server:** Disable
- **Guest Network → DHCP:** Disable (or disable guest network entirely)
- **IoT/Smart Home → DHCP:** Disable
- **Any VLAN scopes:** Disable
- **UPnP:** Disable
- **IPv6 / DHCPv6:** Disable for now

A guest network with its own DHCP server on the same broadcast domain
causes intermittent disconnects, IP conflicts, and "domain controller
could not be contacted" errors. Classic homelab + interview question.

---

## Detect Rogue DHCP

From the DC after role installed:

```cmd
dhcploc 192.168.1.2
```

From any Linux box on LAN (e.g., Pi):

```bash
sudo nmap --script broadcast-dhcp-discover
```

Should show only `192.168.1.2` responding. More than one = duplicate
DHCP servers fighting.

# Group Policy

# GPOs Deployed — homelab.local

**Status:** ✅ 3 custom GPOs deployed and verified
**Naming convention:** `PUBMIX-{Scope}-{Purpose}`
**Last Updated:** 2026-04-21

---

## Overview

Custom GPOs deployed for the lab. Default Domain Policy and Default
Domain Controllers Policy are not modified. All custom GPOs use the
`PUBMIX-` prefix for easy identification in GPMC.

---

## GPO Lifecycle (Standard 5-Step Pattern)

1. **Create** in `Group Policy Objects` container
2. **Edit** — configure settings under Computer or User Config
3. **Link** to the OU where it should apply
4. **Scope** — adjust security filtering if needed (default = Authenticated Users)
5. **Test** with `gpupdate /force` and `gpresult /h`

---

## GPO 1 — `PUBMIX-Workstations-ScreenLock`

**Linked to:** `PUBMIX`
**Status:** ✅ Verified working on `Eric Del Angel-U`

### Settings

User Configuration → Policies → Administrative Templates →
Control Panel → Personalization:

| Setting | Value |
|---|---|
| Enable screen saver | Enabled |
| Force specific screen saver | Enabled, `scrnsave.scr` |
| Password protect the screen saver | Enabled |
| Screen saver timeout | Enabled, `600` seconds (10 min) |

### Verification on client

```powershell
gpupdate /force
gpresult /r /scope:user
```

Output confirmed `PUBMIX-Workstations-ScreenLock` under Applied GPOs.

Registry confirmation:

```powershell
reg query "HKCU\Software\Policies\Microsoft\Windows\Control Panel\Desktop"
```

Output:

```
ScreenSaveActive       REG_SZ    1
ScreenSaverIsSecure    REG_SZ    1
ScreenSaveTimeOut      REG_SZ    600
SCRNSAVE.EXE           REG_SZ    scrnsave.scr
```

---

## GPO 2 — `PUBMIX-Users-DriveMap-K`

**Linked to:** `PUBMIX`
**Purpose:** Map `K:` drive to shared folder for all users
**Status:** ⚠️ Created — file server share is the staging
"PUBMIX Shared" share (see File Services page; `PubmixShared`
recommended without space)

### Configuration

User Configuration → Preferences → Windows Settings → Drive Maps →
New → Mapped Drive:

| Field | Value |
|---|---|
| Action | Update |
| Location | `\\WIN2022\PubmixShared` (or `\\WIN2022\PUBMIX Shared`) |
| Reconnect | ✅ |
| Label as | `PUBMIX Shared` |
| Drive Letter | Use → `K` |
| Hide/Show this drive | Show |

---

## GPO 3 — `PUBMIX-Users-WallpaperSlideshow`

**Linked to:** `PUBMIX`
**Purpose:** Rotating desktop wallpaper for all users
**Wallpapers location:** `\\homelab.local\SYSVOL\homelab.local\scripts\Wallpapers\`

### Configuration

Implemented as four Group Policy Preference Registry items.

User Configuration → Preferences → Windows Settings → Registry → New →
Registry Item (one for each):

| Action | Hive | Key Path | Value Name | Type | Data |
|---|---|---|---|---|---|
| Update | HKEY_CURRENT_USER | `Control Panel\Personalization\Desktop Slideshow` | `Interval` | REG_DWORD | `60000` (1 min) or `600000` (10 min, recommended) |
| Update | HKEY_CURRENT_USER | `Control Panel\Personalization\Desktop Slideshow` | `ImagesRootPath` | REG_SZ | `\\homelab.local\SYSVOL\homelab.local\scripts\Wallpapers` |
| Update | HKEY_CURRENT_USER | `Control Panel\Personalization\Desktop Slideshow` | `Shuffle` | REG_DWORD | `1` |
| Update | HKEY_CURRENT_USER | `Control Panel\Desktop` | `WallpaperStyle` | REG_SZ | `10` (fill) |

If a user manually set a static wallpaper before, log off/on twice
for slideshow setting to take.

---

## GPO Processing — LSDOU Order

1. **L**ocal — local computer policy
2. **S**ite — AD site (rarely used in single-site labs)
3. **D**omain — entire domain (Default Domain Policy)
4. **O**rganizational **U**nit — top-down, parent first

Last writer wins. OU-level GPOs override domain-level.

---

## Useful Commands

```powershell
# Force refresh
gpupdate /force

# Report applied GPOs (HTML — open in browser)
gpresult /h C:\gp-report.html

# Quick text version
gpresult /r /scope:user
gpresult /r /scope:computer

# Generate full GPO inventory
Get-GPOReport -All -ReportType HTML -Path C:\all-gpos.html
```

---

## TODO — Future GPOs

Not yet deployed:

- [ ] `PUBMIX-Baseline-PasswordPolicy` (domain-root linked)
- [ ] `PUBMIX-Workstations-Security-Baseline`
- [ ] `PUBMIX-Audit-Policy-Baseline`
- [ ] `_Admin-Tier0-RestrictLogon` (the GPO that makes the tier
      model actually enforce)
- [ ] `PUBMIX-Workstations-Defender`
- [ ] `Department-targeted` drive mappings using Item-Level Targeting

# Clients

# Domain Join — Windows 11 Pro

**Status:** ✅ Joined
**Domain:** `homelab.local`
**DC:** PCLAB (`192.168.1.2`)
**Test user verified:** `Eric Del Angel-U` (CN under
`PUBMIX/USA/CA/Ventura/Users`)
**Last Updated:** 2026-04-21

---

## Overview

Windows 11 Pro VM joined to `homelab.local`. Domain joins fail more
often from DNS misconfiguration than anything else — DNS must point
to the DC, not the router or a public resolver.

**Windows 11 Home cannot domain-join** — Pro / Education / Enterprise
required.

---

## Pre-flight Checklist

### 1. DNS pointing at the DC (most common gotcha)

In File Explorer, type in the address bar:

```
\\homelab.local
```

If `SYSVOL` and `NETLOGON` folders open → DNS is good. If it errors →
fix DNS before proceeding.

GUI: Network & Internet → Ethernet → DNS server assignment → Edit →
Manual → IPv4 on → Preferred DNS = `192.168.1.2`.

PowerShell:

```powershell
Get-NetAdapter | Where-Object Status -eq "Up" |
  Set-DnsClientServerAddress -ServerAddresses ("192.168.1.2")

nslookup homelab.local
nslookup _ldap._tcp.dc._msdcs.homelab.local
```

Second one must return PCLAB.

### 2. Time sync (Kerberos requirement)

If client and DC clocks drift more than 5 minutes apart, Kerberos
fails and join is rejected.

```powershell
w32tm /resync
```

### 3. Confirm hostname

```powershell
hostname
```

Rename **before** joining if needed. Renaming after join is messy.

---

## Join the Domain

### GUI

1. `Win + R` → `sysdm.cpl` → Enter
2. **Computer Name** tab → **Change...**
3. **Member of:** Domain → `homelab.local` → OK
4. Credentials: `homelab\Administrator` (or any admin)
5. "Welcome to the homelab.local domain" → OK
6. Restart when prompted

### PowerShell (alternative)

```powershell
Add-Computer -DomainName "homelab.local" `
  -Credential (Get-Credential) -Restart
```

---

## Post-Join Validation

After reboot, log in as a domain user (e.g., `homelab\Eric Del Angel-U`).

```powershell
whoami /fqdn
nltest /dsgetdc:homelab.local
gpresult /r /scope:user
```

Expected: GPOs listed (`PUBMIX-Workstations-ScreenLock`, etc.) under
"Applied Group Policy Objects."

---

## Move Computer Object to Correct OU

After join, the computer object lands in the default `Computers`
container. Move it so workstation-targeted GPOs apply.

ADUC on DC:

1. `dsa.msc` → expand `homelab.local` → **Computers** container
2. Find Win11 machine
3. Right-click → **Move...**
4. Navigate to `PUBMIX/USA/CA/Ventura/Workstations` → OK

(Or set up `redircmp` once so future joins land there automatically —
see OU Structure page.)

---

## Common Failure: "AD DC could not be contacted"

99% of the time = DNS. Confirm `\\homelab.local` opens in File
Explorer **before** retrying join.

---

## Verify GPO Applied (Test from Win11 Client)

```powershell
gpresult /r /scope:user
```

Output excerpt confirming `Eric Del Angel-U`:

```
CN=Eric Del Angel-U,OU=Users,OU=Ventura,OU=CA,OU=USA,OU=PUBMIX,DC=homelab,DC=local

Applied Group Policy Objects
    PUBMIX-Workstations-ScreenLock

Group Policy was applied from: PCLAB.homelab.local
```

Registry confirmation:

```powershell
reg query "HKCU\Software\Policies\Microsoft\Windows\Control Panel\Desktop"
```

# File Services

# File Server — Status & Plan

**Status:** ⚠️ Partial — `PUBMIX Shared` SMB share exists, FS-FileServer
role not yet installed
**Host (planned):** WIN2022 (VM 104) — currently the DC, may move to
dedicated VM later
**Share path:** `K:\PUBMIX Shared` → `\\WIN2022\PUBMIX Shared`
**Recommended share alias:** `\\WIN2022\PubmixShared` (no space)
**Last Updated:** 2026-04-21

---

## Overview

A K: drive on WIN2022 holds the `PUBMIX Shared` SMB share for
domain-wide use. The share is mapped to clients via GPO
(`PUBMIX-Users-DriveMap-K`). The full File Server role and per-
department shares were planned but paused when scope changed to a
single shared drive for everyone.

---

## Current State

- ✅ Physical/virtual K: drive mounted on WIN2022
- ✅ SMB share `PUBMIX Shared` exists at `\\WIN2022\PUBMIX Shared`
- ✅ GPO `PUBMIX-Users-DriveMap-K` linked at `PUBMIX`
- ⚠️ File Server role not formally installed (SMB built-in is doing
  the work)
- ⚠️ Share name has a space — causes UNC path / scripting headaches

---

## Recommended Cleanup — Add Spaceless Share Alias

Run on WIN2022:

```powershell
New-SmbShare -Name "PubmixShared" -Path "K:\PUBMIX Shared" `
  -ChangeAccess "Domain Users" `
  -FullAccess "Domain Admins"
```

This adds a second name pointing to the same folder. Update the GPO
drive map to use `\\WIN2022\PubmixShared` going forward.

---

## To Make This a Proper File Server

1. Install role:

```powershell
   Install-WindowsFeature FS-FileServer -IncludeManagementTools
```

2. Optional: install DFS for namespace abstraction:

```powershell
   Install-WindowsFeature FS-DFS-Namespace, FS-DFS-Replication `
     -IncludeManagementTools
```

3. Move file shares to a dedicated VM (best practice — DCs shouldn't
   host user data).

---

## Permissions Layered Model

Two layers must both be configured. The **more restrictive** of the
two wins per user.

| Layer | Where | Tool |
|---|---|---|
| Share permissions | Network-level access | Right-click folder → Properties → Sharing |
| NTFS permissions | File system ACLs | Right-click folder → Properties → Security |

Standard practice: Share = `Authenticated Users: Full Control`, then
let NTFS handle access control.

---

## Original Per-Department Plan (Paused)

For reference if scope expands later:

```
File server (WIN2022):
  C:\Shares\Departments\
    ├── Sales\          → \\WIN2022\Sales$        → H: for Sales-Ventura
    ├── Marketing\      → \\WIN2022\Marketing$    → H: for Marketing-Ventura
    ├── Engineering\    → \\WIN2022\Engineering$  → H: for Engineer-Ventura
    └── Finance\        → \\WIN2022\Finance$      → H: for Finance-Ventura
  C:\Shares\Company\    → \\WIN2022\Company       → S: for everyone
```

`$` suffix on share names = hidden share; users can't browse, only
mount directly. GPO Preferences with **Item-Level Targeting** would
filter each H: drive map by department security group membership.

---

## SYSVOL Wallpaper Folder

Used by `PUBMIX-Users-WallpaperSlideshow` GPO.

Path: `\\homelab.local\SYSVOL\homelab.local\scripts\Wallpapers\`

Drop 3–5 corporate wallpapers (jpg/png) here. SYSVOL replicates
automatically and is readable by all authenticated users — standard
location for company-wide wallpaper deployment.

# Operations

# Proxmox Backups & Snapshots

**Status:** ✅ Manual backups completed
**Backup storage:** `tank-backup` (`/tank/backups/dump/`)
**Snapshot storage:** native ZFS on `tank`
**Last Updated:** 2026-04-21

---

## Overview

Backups vs. snapshots are different in Proxmox:

- **Snapshot** = point-in-time state, stored with the disk on `tank`.
  Fast, copy-on-write, free on ZFS. Best for rollback before risky
  changes.
- **Backup** = full VM export written to a separate storage target
  (`tank-backup`). Heavier but portable and restorable to a different
  VMID.

---

## Manual Backup — All VMs

Single command — handles running and stopped VMs:

```bash
vzdump 100 101 102 104 \
  --storage tank-backup \
  --mode snapshot \
  --compress zstd \
  --notes-template '{{guestname}} manual backup'
```

Notes-template variables supported on Proxmox 6.17.13-2:
`{{cluster}}`, `{{guestname}}`, `{{node}}`, `{{vmid}}`. The
`{{now}}` variable is **not** supported on this version — file name
includes timestamp anyway.

Compressed files appear in:

```bash
ls -lh /tank/backups/dump/
# vzdump-qemu-100-2026_04_21-12_04_48.vma.zst, etc.
```

---

## Backup Duration (Reference)

For 250 GB Win11 disk on `tank`: ~100–120 min at ~31–35 MB/s
(zstd-bound on this hardware). Plan ~3–5 hours for the full 4-VM
sequential job.

---

## Snapshots — VM-Level (GUI Tracked)

CLI:

```bash
qm snapshot 104 DC-DNS-Working-2026-04-20 \
  --description "DC has working DNS forwarders, pre-DHCP install" \
  --vmstate 1

qm listsnapshot 104
qm rollback 104 DC-DNS-Working-2026-04-20
qm delsnapshot 104 OLD-NAME
qm config 104
```

`--vmstate 1` includes RAM (resume to exact moment).
`--vmstate 0` is disk only (faster, smaller, but VM reboots on restore).

GUI: VM → **Snapshots** → **Take Snapshot** → ✅ Include RAM.

---

## ZFS-Native Snapshots — Avoid

```bash
zfs snapshot tank/vm-104-disk-0@name   # works but bypasses Proxmox tracking
```

Won't appear in Proxmox UI. Stick with `qm snapshot` unless explicitly
managing at the ZFS layer.

---

## Snapshot Naming Convention

`{role}-{state}-{YYYY-MM-DD}`

Examples:
- `DC-DNS-Working-2026-04-20`
- `DC-Pre-DHCP-Install-2026-04-20`
- `DC-Post-DHCP-Working-2026-04-21`
- `Win11-Pre-DomainJoin-2026-04-21`
- `Win11-Post-DomainJoin-2026-04-21`

---

## Snapshot Hygiene

- Take a snapshot **before** every major change
- Delete snapshots once the next stable state is verified
- Keep 3–5 per VM max — Proxmox slows with too many
- Snapshots with RAM use noticeable disk space; clean up regularly

---

## Backup Retention

⚠️ **Current retention: 14 days** — confirmed during the product-key
recovery incident on SERVER-LAB. Backups expire automatically on a
14-day window. Consider increasing for build/baseline artifacts.

---

## Lessons Learned (from key recovery incident)

`slmgr /cpky` zeroes the `DigitalProductId` registry value as a
hardening control. Without a pre-cpky snapshot or backup, the original
Retail product key is unrecoverable from the machine. Action items:

- [ ] Document keys in a password vault **before** applying `cpky`
- [ ] Extend backup retention beyond 14 days for baseline build images
- [ ] Take a Proxmox snapshot of the source ISO storage occasionally

# External Lab (separate scope)

# SERVER-LAB — Physical Lab (External)

**Status:** Documented for reference — not part of `homelab.local`
**Hardware:** Dell OptiPlex 5070, Intel i7-9700, NIC Intel I219-V
**OS:** Windows Server 2022 Standard (Retail channel, Build 20348.587)
**Hostname:** SERVER-LAB
**Use:** Hyper-V host for nested lab VM `WINSERVER2`
**Last Updated:** 2026-04-22

---

## Overview

Separate from the home Proxmox lab (`homelab.local` / `pve`).
SERVER-LAB is a physical OptiPlex 5070 used as a Windows Server
2022 / Hyper-V host. Documented here for completeness but is not part
of the `homelab.local` domain.

This page captures the issues and resolutions encountered, since they
recur on similar hardware.

---

## Issue 1 — NIC Not Detected on Fresh Install

**Cause:** Windows Server 2022 ships with fewer built-in NIC drivers
than client Windows. Onboard Intel I219-V is detected as unknown.

### Driver source

Intel "Wired Driver" complete pack supports the I219 family on
Server 2022:

```
Release_31.1\PRO1000\Winx64\WS2022\
```

INF file: `e1d68x64.inf`

### Catch — I219-V is server-blocked

Intel's INF explicitly refuses to install the V variant on Server OS
(consumer part). Workaround: install the **I219-LM** driver against
the V hardware. Same silicon, different PCI ID, driver is signed and
works.

### Install procedure

1. Device Manager → unknown Ethernet Controller → Update driver
2. **Browse my computer for drivers**
3. **Let me pick from a list of available drivers on my computer**
4. **Have Disk** → browse to `e1d68x64.inf` in the WS2022 folder
5. **Uncheck** "Show compatible hardware"
6. Pick the highest-numbered `Intel(R) Ethernet Connection (X) I219-LM`
7. Accept the compatibility warning → install
8. Reboot

### Verify

```powershell
Get-NetAdapter | Format-List Name, InterfaceDescription, DriverVersion, Status, LinkSpeed
```

Expected: `Intel(R) Ethernet Connection ... I219-LM`, Status `Up`,
driver version ~14.x.

---

## Issue 2 — Hyper-V Refused to Install Inside a VM

`Install-WindowsFeature -Name Hyper-V` failed inside `WINSERVER2`
(a VM on SERVER-LAB) with:

```
Hyper-V cannot be installed: The processor does not have required
virtualization capabilities.
```

### Root cause

Nested virtualization not enabled. Host (SERVER-LAB) had not exposed
virtualization extensions to the guest VM.

### Fix — run on the HOST (not the guest)

```powershell
# Stop the guest first
Stop-VM -Name "WINSERVER2"

# Expose virtualization extensions
Set-VMProcessor -VMName "WINSERVER2" -ExposeVirtualizationExtensions $true

# Verify
Get-VMProcessor -VMName "WINSERVER2" |
  Select VMName, ExposeVirtualizationExtensions

Start-VM -Name "WINSERVER2"
```

Then inside WINSERVER2, retry:

```powershell
Install-WindowsFeature -Name Hyper-V, Hyper-V-PowerShell `
  -IncludeManagementTools -Restart
```

---

## Issue 3 — Product Key Recovery Incident

Server activated via Retail channel (partial key `QV89Y`). Hardening
script ran `slmgr /cpky`, which zeroes `DigitalProductId` in the
registry. Key was no longer recoverable from the machine.

### What was confirmed

- Registry: `DigitalProductId` bytes 52–66 zeroed (cpky did its job)
- Setup logs (`C:\Windows\Panther\setupact.log`) referenced
  `pid.txt` / `ei.cfg` from install media — key was on the original
  USB/ISO, not on disk
- Proxmox snapshots: none from before hardening
- Proxmox backups: 14-day retention had expired
- No System Restore points
- No `Windows.old` folder

### Resolution path

The only realistic recovery was:

1. Locate the original install ISO with `sources\pid.txt` embedded
2. Buy a replacement key from an authorized partner (~$1,000,
   same-day via email)
3. Have the honest conversation with leadership

### Prevention going forward

- [ ] Document Retail keys in password vault **before** running `cpky`
- [ ] Extend Proxmox backup retention beyond 14 days
- [ ] Keep build ISOs in a permanent archive, not deleted after install
- [ ] Tag baseline snapshots as "do not auto-prune"

### What `slmgr /cpky` actually does

Zeroes the encoded product key bytes in the registry. By design,
**no recovery tool** (ProduKey, ShowKeyPlus, registry decoders)
can recover the key after this — that's the entire point of the
command. It's a defensible security control.

---

## Reference — slmgr Diagnostics

```cmd
slmgr /dlv
```

Key fields:
- **Description:** channel (RETAIL / VOLUME_KMSCLIENT / VOLUME_MAK)
- **Partial Product Key:** last 5 chars only (by design)
- **License Status:** Licensed / Notification / Grace
- **Notification Reason:** error code if not activated