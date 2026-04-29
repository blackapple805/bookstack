# DC Cross-Site Restore — Hyper-V (External)

Cross-site disaster-recovery exercise: restoring the questlab.local domain controller (SERVER-QUEST) onto a friend's Hyper-V host using a Windows Server Backup image carried on a physical drive. Physical disk passthrough + System Image Recovery from boot ISO. April 2026.

# Cross-Site Restore

# Prep & Hyper-V VM Setup at Remote Site

**Status:** ✅ Restore completed  
**Source DC:** SERVER-QUEST (`questlab.local`)  
**Backup carrier:** WD MyPassport (NTFS, 931 GB) — same drive used  
for the original `wbadmin` system image  
**Target host:** Friend's Windows Server with Hyper-V role installed  
**Target VM:** New Gen 2 VM running Windows Server 2022 Standard  
**Date:** 2026-04-26 (continuation of the home-site rebuild work)

---

## Overview

Real-world cross-site DR exercise. Walked the original DC's full  
system image (`wbadmin start backup` output, the `WindowsImageBackup`  
folder structure) over to a friend's Hyper-V host on a USB drive,  
attached the drive as a physical disk passthrough to a new  
empty Hyper-V VM, then booted that VM from a Windows Server 2022  
install ISO and used **Repair your computer → Troubleshoot →  
System Image Recovery** to restore SERVER-QUEST onto fresh virtual  
hardware.

This validates the recovery path for the home-site Proxmox migration:  
if the same backup can be restored on someone else's hypervisor,  
restoring it back onto the home Proxmox host is a much smaller risk.

---

## What I Brought

| Item | Contents |
|---|---|
| WD MyPassport (NTFS) | `WindowsImageBackup\SERVER-QUEST\` (full image), `DC-Backup\Exports\` (config exports), `Proxmox\virtio-win.iso` |
| USB stick (FAT32) | Windows Server 2022 Standard eval ISO from Microsoft Evaluation Center |
| Phone | `ipconfig /all` screenshot, OU CSV, GPO list — read-only reference |

---

## Step 1 — Create the Empty Hyper-V VM (Gen 2)

On the friend's Hyper-V host, **Hyper-V Manager → New → Virtual Machine**:

| Setting | Value |
|---|---|
| Name | `DC-Restore-Test` |
| Generation | **Generation 2** (UEFI, Secure Boot capable) |
| Memory | 4096 MB (Dynamic Memory: off) |
| Network | **Not Connected** initially — keeps the restored DC off the friend's LAN until verified |
| Virtual hard disk | New, 150 GB, default location |
| Install OS from | Bootable image file → `Windows Server 2022 Eval` ISO |

Generation 2 matters: matches the source DC's UEFI boot mode and
SCSI disk topology. Restoring a UEFI-installed Windows onto a Gen 1
(BIOS) VM fails at boot.

![Recovery Wizard](./img/recovery-wizard.jpg)
*Figure 1: Initializing the Windows Recovery Environment (WinRE) restore wizard.*

After creation, **don't start the VM yet** — hardware needs more changes.

---

## Step 2 — VM Settings: Disable Secure Boot Temporarily

Right-click VM → **Settings**:

- **Security:**
  - **Secure Boot:** ✅ Enabled
  - **Template:** `Microsoft Windows`
- **Processor:** 2 virtual processors (more is fine, 2 minimum)
- **Integration Services:** ✅ check **Guest services**



Guest Services is off by default. Turn it on now — useful if extra  
drivers or files need to be pushed in later without networking.

---

## Step 3 — Take the Backup Drive Offline on the Host

This is the step that surprises everyone. Hyper-V can only attach a
physical disk if the **host OS is not using it** — meaning the drive
must be in an offline state in Disk Management.

1. Plug the WD MyPassport into the friend's Windows Server.
2. Wait for the host to detect it (don't browse it in Explorer —
   leave it untouched).
3. Open **Disk Management** (`Win + X` → Disk Management, or
   `diskmgmt.msc`).
4. Locate the MyPassport in the disk list (usually `Disk 2` or `Disk 3`,
   identifiable by the 931 GB size).
5. **Right-click the disk label on the left** (not the partition) →
   **Offline**.

![Excluding Disks](./img/disk-exclusion.png)
*Figure 3: Ensuring the source disk is excluded from formatting.*

The disk now shows as **Offline** with a red down-arrow icon.

> **Why offline:** Hyper-V requires exclusive access to a passthrough
> disk. The host and the VM cannot both read/write to it
> simultaneously, so the host has to release the disk first.
---

## Step 4 — Attach the Offline Disk to the VM (Physical Pass-Through)

Back in Hyper-V Manager → right-click the VM → **Settings**:

1. **SCSI Controller** → **Hard Drive** → **Add**
2. In the new Hard Drive panel:
   - Select **Physical hard disk**
   - Drop-down lists every offline physical disk on the host
   - Pick the WD MyPassport (size = 931 GB)
3. **Apply** → **OK**

The VM now has two virtual storage devices:

| Device | Backing | Role |
|---|---|---|
| Hard Drive 1 | 150 GB VHDX (`DC-Restore-Test.vhdx`) | Empty — restore target |
| Hard Drive 2 | Physical disk (WD MyPassport) | Read-only source — holds `WindowsImageBackup\` |
| DVD Drive | Server 2022 eval ISO | Boot media for recovery environment |

![Restore Success](./img/restore-success.png)
*Figure 4: Successful completion of the Bare Metal Recovery process.*

If "Physical hard disk" shows as greyed out or no disks appear, the
WD MyPassport is not offline yet. Go back to Step 3.

---

## Step 5 — Pre-Boot Check

Before powering on:

- VM has 4+ GB RAM
- Network adapter is **Not Connected** (or attached to an isolated
  internal switch with no DHCP / no internet) — important so the
  restored DC doesn't try to register itself in the friend's DNS
  or replicate AD changes back to the home network
- DVD drive has the Windows Server 2022 ISO loaded
- BIOS/Firmware boot order in VM Settings → **Firmware:**
  put `DVD Drive` above `Hard Drive` so the VM boots from ISO

Take a Hyper-V checkpoint of the VM here before pressing Start.
If the restore goes sideways, rolling back is one click.

Move to **Page 2 — Restore via System Image Recovery**.

# Restore via System Image Recovery

**Status:** ✅ Completed successfully
**Restore time:** ~45 minutes for the OS volume (~75 GB used) over
USB 3.0 to local VHDX
**Last Updated:** 2026-04-26

---

## Overview

This page covers the actual restore: boot the empty Hyper-V VM from
the Windows Server 2022 ISO, jump into the recovery environment, and
point System Image Recovery at the `WindowsImageBackup` folder on the
attached pass-through disk.

The procedure matches Microsoft's documented Bare Metal Recovery flow
and the well-known Timothy Gruber blog post on Hyper-V WSB restores —
this is the standard Microsoft-supported path, not a hack.

---

## Step 1 — Boot the VM from ISO

In Hyper-V Manager: right-click VM → **Start**, then immediately
**Connect** to open the console.

When the VM posts, it shows:

```
Press any key to boot from CD or DVD....
```

Press a key fast (this prompt times out in ~5 seconds). Miss it and
the VM tries to boot the empty 150 GB VHDX, fails, and you have to
power-cycle.

The Windows Server 2022 setup splash appears, then the language
selection screen.

---

## Step 2 — Reach the Recovery Environment

On the language screen:

1. Click **Next** (defaults are fine)
2. **Don't click "Install now"** — instead, click **Repair your
   computer** at the bottom-left
3. **Choose an option** screen → **Troubleshoot**
4. **Advanced options** → **System Image Recovery**

This launches the Windows Recovery Environment's image-restore wizard.

---

## Step 3 — Select the Backup Image

The wizard scans for available system images.

### If it auto-detects the backup

A dialog shows:
> **Re-image your computer** — Windows found a system image on this
> computer.

Click **Use the latest available system image (recommended)** →
**Next**.

Verify the date/time matches the home-site `wbadmin` run. If it does,
proceed. If it shows an older or unexpected backup, switch to manual
selection.

### If auto-detection fails

Click **Select a system image** → **Next** → wizard lists detected
backup volumes. The pass-through disk (WD MyPassport) appears with
its `WindowsImageBackup\SERVER-QUEST\` content.

If the wizard says "No system image found":

- The pass-through disk isn't attached correctly — power off, re-check
  Disk Management offline status and SCSI controller assignment
- The backup folder is on a sub-folder (must be at the **root** of
  the drive: `D:\WindowsImageBackup\SERVER-QUEST\`, not nested deeper)
- The image was created on incompatible hardware family — won't
  apply here, since both ends are Windows Server 2022 x64

---

## Step 4 — Confirm Restore Settings

| Option | Setting |
|---|---|
| **Format and repartition disks** | ✅ checked — wipes the empty 150 GB VHDX before restore |
| **Exclude disks** | ✅ Click and **exclude the WD MyPassport itself** so the wizard doesn't try to format the source drive |
| **Install drivers** | Not needed — both source and target are virtualized Windows Server 2022 |
| **Advanced** | Leave default (auto-restart, check for errors) |

⚠️ **The "Exclude disks" step is critical.** If the source backup
drive is not excluded, the wizard can offer to format it, which would
nuke the only copy of the backup mid-restore. Always exclude the
source.

Click **Next** → **Finish** → confirm "Yes" on the final
**All data on the drives to be restored will be replaced** warning.

---

## Step 5 — Wait

Restore progress shows percentage complete. For ~75 GB used on
USB 3.0 → local SCSI VHDX: ~30–60 minutes.

Don't:
- Touch the VM console
- Interact with Disk Management on the host
- Disconnect the WD MyPassport
- Pause the VM

When the restore finishes, the wizard reboots the VM automatically.

---

## Step 6 — First Boot of Restored DC

The VM reboots into the freshly-restored Windows Server 2022 — same
hostname (`SERVER-QUEST`), same `questlab.local` domain, same AD
database, same SYSVOL, same DHCP scope, same GPOs.

### Critical: keep the network adapter disconnected on first boot

The restored DC believes it is still SERVER-QUEST at `192.168.1.2`
on the home LAN. If the VM is connected to the friend's network and
their DHCP hands it a different IP, AD/DNS gets confused and tries
to replicate state to a partner that doesn't exist.

Leave the NIC **Not Connected** until the system has fully booted
and you have confirmed:

```powershell
hostname                          # SERVER-QUEST
(Get-ADDomain).DNSRoot            # questlab.local
Get-Service NTDS, DNS, DHCPServer # all Running
dcdiag /test:replications         # expect failures — that's fine, no partner present
```

Empty replication queue + services running = restore is healthy.

---

## Step 7 — Detach Source, Promote to "Working"

Once verified:

1. Power off the VM
2. Hyper-V Manager → VM Settings → SCSI Controller → select the
   **Physical hard disk** entry → **Remove** → Apply
3. On the host: Disk Management → right-click WD MyPassport →
   **Online** → eject safely
4. Optional: now that the restored DC is booting cleanly off its own
   VHDX, attach it to an isolated Hyper-V internal switch for further
   testing without touching the friend's production LAN

---

## Validation Checklist

- [x] VM boots cleanly from the new VHDX with no source drive attached
- [x] Active Directory database mounts (`ntds.dit` healthy)
- [x] DNS service starts and resolves `questlab.local`
- [x] DHCP scope is intact (`Get-DhcpServerv4Scope`)
- [x] All 13 GPOs visible in GPMC
- [x] OUs match home-site structure (PUBMIX, _Admin, Disabled)
- [x] AD users present (`Get-ADUser -Filter *`)

---

## Key Lessons

**Disk passthrough is the right tool for this.** Trying to mount the
WD MyPassport as a network share from inside the recovery environment
would have required loading network drivers in WinRE, configuring an
IP, and authenticating to the share. The Gen 1 VM trick of using a
"Legacy Network Adapter" doesn't apply on Gen 2 VMs at all. Direct
pass-through is far simpler.

**Generation 2 VM matters.** UEFI source → Gen 1 (BIOS) target =
unbootable. Always match firmware mode to the source.

**Network must stay disconnected on first boot.** A restored DC
attached to a foreign LAN can corrupt its own AD replication metadata
trying to find partners that don't exist.

**Guest Services should be enabled before first boot.** Allows the
Hyper-V host to push files into the VM via `Copy-VMFile` without
needing networking — useful if a missing driver needs to go in.

**The full backup chain is now validated end-to-end:**
`wbadmin start backup` on bare metal → physical drive transport →
Hyper-V VM → System Image Recovery → working DC. The home-site
Proxmox restore can proceed with confidence.

---

## References

- Microsoft Learn: pass-through disk requires offline state on host;
  Gen 1 and Gen 2 VMs both supported
- Timothy Gruber's blog: documents the same exact restore procedure
  for Hyper-V WSB images using "Repair → Troubleshoot →
  System Image Recovery"
- Microsoft Evaluation Center: Windows Server 2022 ISO