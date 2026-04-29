# 📚 Bookstack Infrastructure & Recovery Lab

This repository contains the configuration for a Bookstack environment and detailed disaster recovery runbooks for Windows Server 2022.

## 📖 Project Documentation
* **[Infrastructure Overview](./docs/homelab-infrastructure.md)**: Details on Proxmox storage, ZFS pools, and ISO management.
* **[Disaster Recovery Guide](./docs/dc-cross-site-restore-hyper-v-external.md)**: A step-by-step validated restoration of a Domain Controller via System Image.

## 🚀 Environment Setup
To deploy the Bookstack wiki stack:
```bash
docker-compose up -d
```

---
*Last Updated: April 2026*
