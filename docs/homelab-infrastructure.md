# Homelab Infrastructure: server-lab

> **Host Architecture:** A single Proxmox VE node virtualizing a complete Windows enterprise environment. The core of the network relies on a Windows Server 2022 instance managing AD DS, DNS, DHCP, and GPOs.

**Owner:** Eric Del Angel · **Hypervisor:** Proxmox VE 9.1.7  
**Domain:** `homelab.local` · **Host Node:** `server-lab`  
**Primary DC:** PCLAB (`192.168.1.2`) · **Last Updated:** 2026-04-22

---

## Environment Summary

This infrastructure centralizes all mission-critical services onto a single **Windows Server 2022 Standard Edition** instance (`PCLAB`). By hosting the Domain Controller, Windows/Linux clients, and core networking services on one physical Proxmox host, the environment maintains a small footprint while providing full enterprise-grade management via Active Directory.

---

## Hardware & Storage

> **Drive Health:** The 2 TB Seagate Barracuda is aged (~6.4 years) with 760 reallocated sectors.
> **Policy:** Scratch/Lab storage only. **No high-value data without external backups.**

### ZFS Pool: `tank`

| Storage ID      | Type | Content                    | Configuration                    |
| :-------------- | :--- | :------------------------- | :------------------------------- |
| `tank`          | ZFS  | Disk images, Containers    | `lz4`, `ashift=12`, `atime=off`  |
| `tank-iso`      | Dir  | ISO installation media     | —                                |
| `tank-backup`   | Dir  | VZDump backup files        | —                                |
| `tank-snippets` | Dir  | Cloud-init config snippets | —                                |

### VM Inventory

| VMID    | Hostname   | Role                          | OS                      |
| :------ | :--------- | :---------------------------- | :---------------------- |
| **104** | **PCLAB**  | **Primary Domain Controller** | **Windows Server 2022** |
| 100     | Win11PRO   | Primary Windows Workstation   | Windows 11 Pro          |
| 101     | u-server   | Linux Infrastructure Support  | Ubuntu 24.04            |
| 103     | Win11PRO-2 | Secondary Testing Client      | Windows 11 Pro          |

---

## Active Directory Structure

*The following tree uses a Two-Pattern OU design for administrative tiering and regional segmentation.*

```text
homelab.local
├── _Admin                         (Tiered Administration)
│   ├── Tier0-Identity             (DC Admins, Identity Groups)
│   ├── Tier1-Servers              (Server Admins, Member Servers)
│   └── Tier2-Workstations         (Support Staff, Workstations)
├── PUBMIX                         (Production Assets)
│   ├── USA
│   │   └── CA
│   │       └── Ventura            (Users, Computers, Groups)
│   └── CHILE
│       └── Valparaiso             (Users, Computers, Groups)
└── Disabled                       (Staging for Deletion; GPO Blocked)
```