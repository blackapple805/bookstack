# 📚 Bookstack Infrastructure & Recovery Lab

![Status](https://img.shields.io/badge/Status-Stable-success)
![Platform](https://img.shields.io/badge/Platform-Proxmox%20%7C%20Docker-blue)
![DR](https://img.shields.io/badge/DR-Validated-orange)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

This repository serves as a functional documentation-first homelab environment. It bridges the gap between **Documentation-as-Service** (Bookstack) and **Enterprise Continuity** (Windows Server Disaster Recovery).

---

## 🎯 Project Goals
- **Centralized Knowledge:** Hosting a robust Wiki engine (Bookstack) for homelab runbooks.
- **Enterprise Continuity:** Validating Bare Metal Recovery (BMR) for Active Directory Domain Controllers across different hypervisors.
- **Infrastructure-as-Code:** Utilizing Docker Compose for rapid service deployment and portability.

## 🛠️ Technology Stack

| Category | Technology | Purpose |
| :--- | :--- | :--- |
| **Hypervisor** | Proxmox VE | Primary host for ZFS storage and container orchestration. |
| **Containerization** | Docker / Compose | Deployment of the Bookstack application and database. |
| **Directory Services** | Windows Server 2022 | AD/DNS services for the `questlab.local` domain. |
| **Storage** | ZFS (tank) | High-integrity storage for ISOs and virtual disks. |
| **DR Platform** | Microsoft Hyper-V | Remote site target for cross-platform restoration exercises. |

## 🗺️ Lab Architecture & Workflow
The lab follows a hybrid lifecycle where documentation is both the **output** and the **subject** of the recovery exercises.



1. **Deployment:** Docker Compose initializes the Bookstack wiki.
2. **Infrastructure:** Proxmox manages the underlying ZFS pools and ISO libraries.
3. **Recovery:** A physical backup of a Domain Controller is restored to an external Hyper-V host via disk passthrough.

## 📖 Documentation Index
Detailed technical runbooks are maintained in the `/docs` directory:

| Resource | Technical Focus |
| :--- | :--- |
| 🏗️ **[Infrastructure Overview](./docs/homelab-infrastructure.md)** | ZFS configuration, storage hierarchies, and Proxmox management. |
| 🛡️ **[Disaster Recovery Guide](./docs/dc-cross-site-restore-hyper-v-external.md)** | Full walkthrough of the BMR process on Hyper-V using passthrough storage. |

## 🚀 Quick Start
To spin up the Bookstack environment locally:

```bash
# Clone the repository
git clone [https://github.com/blackapple805/bookstack.git](https://github.com/blackapple805/bookstack.git)

# Start the stack
cd bookstack
docker-compose up -d