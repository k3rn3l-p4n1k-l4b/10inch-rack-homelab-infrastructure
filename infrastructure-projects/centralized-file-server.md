# Centralized File Server & Storage Configuration (TrueNAS Core)

This document covers the implementation of a centralized storage solution using **TrueNAS Core** to serve network shares, manage ZFS datasets, and provide a secure storage target for infrastructure backups.

## 💾 Storage Architecture & ZFS Pool Layout
*   **Storage Pool**: `tank` (ZFS Mirror / RAID-Z1)
*   **Dedicated Dataset**: `tank/proxmox-backups`
*   **Protocol**: NFS (Network File System) / SMB (Server Message Block)

> ![TrueNAS Pool Status](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/Truenas%20screenshot%20jpg.jpg)

## 🔧 TrueNAS Dataset & Share Configuration

### 1. Dataset Creation
*   Compression: `lz4` (Default, optimized for backup efficiency)
*   Sync: `standard`
*   Acl Type: `Generic` (For NFS) / `SMB` (For Windows/Linux cross-compatibility)

### 2. Network Share Settings
The dataset is exported via **NFS** to allow the Proxmox Virtualization Host to connect with minimal overhead and native Linux permissions.
*   **Path**: `/mnt/tank/proxmox-backups`
*   **Authorized Networks**: Restricted to the local management VLAN (e.g., `192.168.X.0/24`)
*   **Maproot User**: `root` (Required for Proxmox backup storage access)

> ![NFS Share Settings](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/Shares%20screenshot%20jpg.jpg)

---
## 🚀 Verification
From the Proxmox CLI or an authorized Linux terminal, network accessibility and mount export rights can be verified using:
```bash
showmount -e <TRUENAS_IP>
```
![Mount Verufucation](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/Proxmox%20show%20mount.jpg)
