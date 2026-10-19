# Files

Where my files live, how they get onto devices, how other machines use them, and how they're kept safe.

```mermaid
flowchart LR
    subgraph sources["Everything that gets backed up"]
        laptops["Laptops"]
        servers["Servers<br/>(Docker host, workstation)"]
        nasdata["NAS data<br/>(photos, files, library)"]
        vms["Proxmox VMs"]
    end
    subgraph siteA["Site A"]
        nasA["Backup NAS A<br/>restic repos"]
        pbsA["PBS A"]
    end
    subgraph siteB["Site B"]
        nasB["Backup NAS B<br/>restic repos"]
        pbsB["PBS B"]
    end
    laptops & servers & nasdata -- restic --> nasA
    laptops & servers & nasdata -- restic --> nasB
    vms -- vzdump --> pbsA
    pbsA -. "PBS sync job" .-> pbsB
```

The NAS holds the primary copy of everything. [Syncthing](sync.md) puts its folders onto devices, and [Samba](samba.md) shares them with apps on other machines. Neither of those is a backup.

| Page | Job |
|---|---|
| [File sync](sync.md) | **Syncthing** keeps the same folders on every device that needs them, with the NAS as the always-on hub. |
| [Samba](samba.md) | **SMB shares** let other machines and apps use the NAS's files without their own copy. |
| [Backups](backups.md) | **restic** for files and **Proxmox Backup Server** for VMs to two NAS boxes at two physical sites. |

Each layer does one job. Sync and shares make files *available*. Only the backups make them *safe*.
