# Proxmox VE Automated Backup Implementation

This project implements an automated, scheduled disaster recovery pipeline that backs up critical Virtual Machines (VMs) and Linux Containers (LXCs) from the **Proxmox VE Host** to the centralized **TrueNAS Core** storage server.

## 🔗 Infrastructure Integration

The backup system connects Proxmox natively to the TrueNAS NFS share using the standard storage APIs.

### 1. Proxmox Storage Link
*   **Storage ID**: `truenas-backup`
*   **Type**: NFS
*   **Server**: `<TRUENAS_IP>`
*   **Export**: `/mnt/tank/proxmox-backups`
*   **Content Type**: `VZDump backup file` (Disables ISO/Disk Image clutter)
*   **Retention Policy**: Max Backups kept per VM/LXC: `3` (Rotated automatically)

> ![Proxmox Storage Target](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/proxmoxstoragescreenshot.jpg)

## 📅 Scheduled Backup Policies (VZDump)

Automated backup jobs run natively via Proxmox's Datacenter scheduling daemon to guarantee business continuity.

### Job Configuration:
*   **Selection**: All critical production nodes and containers
*   **Target Storage**: `truenas-backup`
*   **Schedule**: Daily at `02:00` AM (Using Cron notation: `02:00`)
*   **Compression**: `ZSTD` (Fast, multi-threaded compression algorithm)
*   **Mode**: `Snapshot` (Zero downtime; live backup while VMs are running)

> ![Proxmox Backup Schedule](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/backupschedulescreenshot.jpg)

---

## 🛠️ CLI / Infrastructure as Code Equivalent

For automated deployment or auditing, the storage configuration is natively written directly to Proxmox's underlying storage configuration file at `/etc/pve/storage.cfg`:

```nfs
nfs: truenas-backup
        path /var/lib/vz/snippets/truenas-backup
        server <TRUENAS_IP>
        export /mnt/tank/proxmox-backups
        content backup
        prune-backups keep-last=3
```
> ![Config File Output](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/storageconfigscreenshot.jpg)
