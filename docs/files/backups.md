# Backups

Backups are done with two tools: **[restic](#restic-and-backrest)** for files, and **[Proxmox Backup Server](#proxmox-backups)** for whole VMs and LXC containers. File sync is *not* part of this. [Syncthing](sync.md) puts files on devices, which is a different job.

!!! warning "Three things that are not backups"
    - **Sync** (Syncthing, Dropbox). Delete or encrypt a file on one device and the deletion syncs everywhere within seconds.
    - **Parity or RAID** (SnapRAID, ZFS, mdadm). These survive a dead disk. They don't help with accidental deletion, ransomware, or the whole box dying.
    - **Snapshots on the same machine.** They're gone if the machine is.

    A backup is a separate, versioned copy you can go back in time with, ideally somewhere else. The usual rule is **3-2-1**: three copies, on two different kinds of storage, one of them offsite.

## The goal: two NAS boxes, two sites

Every device is backed up to **two separate NAS boxes at two different physical locations**, joined by [Tailscale](../homelab/index.md#tailscale). Either site can burn down, flood or be burgled, and a full, versioned copy of everything still exists at the other.

```mermaid
flowchart LR
    subgraph sources["Everything that gets backed up"]
        laptops["Laptops"]
        servers["Servers<br/>(Docker host, workstation)"]
        nasdata["NAS data<br/>(photos, files, library)"]
        vms["Proxmox VMs & LXCs"]
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

- **Files go to both sites directly.** Each machine has one restic repository per backup NAS, with the same paths and schedule but run at different times. You could instead back up to one repo and `restic copy` it to the second, but two independent repos mean one corrupted repo can't spread to the other.
- **VMs and LXCs go to one PBS, which syncs to the other.** PBS has built-in [sync jobs](https://pbs.proxmox.com/docs/managing-remotes.html) that pull a datastore from a remote PBS, so the hypervisor only uploads each backup once.
- **Each site's NAS stays independent.** It isn't a mirror of the other: a mistake replicated instantly to both copies is the failure this setup exists to avoid.

| Layer | Tool | Protects against |
|---|---|---|
| Database dumps | [systemd timer + `pg_dumpall`](#database-dumps) | A corrupted or half-written database inside a file-level backup. |
| Files → both NAS boxes | [restic + Backrest](#restic-and-backrest) | Deletion, ransomware, losing a whole site. Versioned back two years. |
| VMs & LXCs → PBS | [Proxmox Backup Server](#proxmox-backups) | A guest's OS disk dying, or a bad upgrade. Restore the whole VM or container in minutes. |
| Backup NAS disks | [mergerfs + SnapRAID](#snapraid-and-mergerfs) | One disk in a backup NAS dying. |

### The night

Jobs are staggered so they don't compete for disks and bandwidth:

| Time (UTC) | Job |
|---|---|
| 00:00 | Database dumps on the Docker host |
| 01:30 | Proxmox backs up all VMs and LXCs to PBS |
| 02:30 | Backrest on the NAS → backup NAS |
| 03:30 | Backrest on the Docker host → backup NAS |
| 05:00 (local) | SnapRAID diff / sync / scrub on the backup NAS |
| weekly | restic `prune` and `check` on each repo |

## Database dumps

Copying a database's data directory while it's running can give you a backup that won't open. So once a night, before the file backups run, a systemd timer dumps every database to compressed SQL files with a date in the name:

```bash title="db_backup.sh"
#!/bin/bash
# Nightly dump of every DB container, run by a systemd timer (as root).
# No passwords: Postgres trusts its local socket, MariaDB reads the root
# password from its own container environment.
set -uo pipefail
BACKUPDIR=${BACKUPDIR:-$HOME/docker/db_backups}
KEEP_DAYS=2
STAMP=$(date +%Y%m%d%H%M)
mkdir -p "$BACKUPDIR"
status=0

dump() {  # container name, then the command to run inside it
  local c=$1 out="$BACKUPDIR/$1-$STAMP.sql.gz"
  shift
  if docker exec "$c" sh -c "$*" | gzip > "$out" && [ "$(gzip -dc "$out" | head -c 1 | wc -c)" -gt 0 ]; then
    echo "ok: $c ($(du -h "$out" | cut -f1))"
  else
    echo "FAILED: $c" >&2; rm -f "$out"; status=1
  fi
}

for c in immich-postgres miniflux-db forgejo-db; do   # every Postgres container
  dump "$c" 'exec pg_dumpall -U "${POSTGRES_USER:-postgres}"'
done
dump mariadb 'exec mariadb-dump -uroot -p"$MYSQL_ROOT_PASSWORD" --all-databases --single-transaction'

find "$BACKUPDIR" -name "*.sql.gz" -mtime +$KEEP_DAYS -delete
exit $status
```

```ini title="/etc/systemd/system/docker-db-backup.timer"
[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

Only two days are kept locally, because the `db_backups/` folder is included in the restic backup, which holds the long history.

Some details:

- **No secrets in the script.** The official Postgres image trusts connections over the local socket *inside* the container, so `docker exec … pg_dump` needs no password. For MariaDB, the single quotes mean `$MYSQL_ROOT_PASSWORD` is expanded inside the container, from its own environment. The script can therefore live in git.
- **`pg_dumpall` / `--all-databases`** dump everything in each container, so the script doesn't need to know any database or user names. Restore a `pg_dumpall` file with `psql -U postgres -f dump.sql`.
- **`--single-transaction`** gives a consistent MariaDB dump without locking tables.
- **Empty dumps count as failures.** A container that's gone or renamed otherwise leaves a 20-byte `.gz` every night that looks like a backup. The non-zero exit status marks the systemd unit as failed, which shows up in monitoring.

## restic and Backrest

[restic](https://restic.net) makes encrypted, deduplicated snapshots. Each run only uploads what changed, but every snapshot is a complete point-in-time copy you can browse or restore from. [Backrest](https://github.com/garethgeorge/backrest) is a web UI and scheduler on top of restic: plans, retention, a browsable snapshot list, and notifications when something fails.

Both the NAS and the Docker host run Backrest in Docker and back up over **SFTP via Tailscale**. Nothing has to be exposed to the internet, and each backup NAS just needs an SSH account. In Backrest, a second site is one more repo and one more plan pointing at it.

| | NAS | Docker host |
|---|---|---|
| Backs up | Personal files, photos (including Immich's library), Lightroom, the shared library | The whole compose repo and service data, DB dumps, books/audiobooks/music |
| Skips | — | Game libraries and other things that can be downloaded again |
| Repository | `sftp:<user>@<backup-nas>:/srv/<pool>/<host>-backup` | same, one repo per host |
| Schedule | 02:30 daily | 03:30 daily |
| Keep | 14 daily, 8 weekly, 12 monthly, 2 yearly | same |
| Maintenance | `prune` weekly (repack when >10 % unused), `check` weekly | same |

The source directories are mounted into the Backrest container read-only:

```yaml title="compose excerpt"
  backrest:
    image: garethgeorge/backrest:latest
    volumes:
      - $DOCKERDIR/backrest/data:/data
      - $DOCKERDIR/backrest/config:/config
      - $TEMPDIR/backrest/cache:/cache      # restic cache, makes runs much faster
      - $DOCKERDIR/backrest/.ssh:/.ssh:ro   # key for the SFTP repository
      - $USERDIR/docker:/backup_data:ro
      - $MEDIADIR:/backup_media:ro
    environment:
      - BACKREST_DATA=/data
      - BACKREST_CONFIG=/config/config.json
      - XDG_CACHE_HOME=/cache
```

!!! danger "Save the repository password somewhere else"
    restic encrypts everything with the repository password. Without it the backup is unreadable, including by you. Keep it in your password manager, and **not only** on the machine being backed up.

What to leave out: anything that can be downloaded again (media from the \*arr stack, game libraries, Docker images, caches). Backing up tens of TB of replaceable media offsite costs more than it's worth. The irreplaceable things, such as photos, documents, config and databases, are a few TB.

Restoring from the command line, without Backrest:

```bash
export RESTIC_REPOSITORY=sftp:<user>@<backup-nas>:/srv/<pool>/<host>-backup
restic snapshots                                    # list
restic ls latest /backup_data/paperless             # browse
restic restore latest --target /tmp/restore --include /backup_data/paperless
restic mount /mnt/restic                            # or browse every snapshot as a folder
```

## Proxmox backups

The hypervisor backs up every guest, both VMs and LXC containers, nightly to a [Proxmox Backup Server](https://www.proxmox.com/en/products/proxmox-backup-server/overview) (PBS) at the second site, over Tailscale. PBS deduplicates across guests and nights, so keeping many restore points costs little space.

```text title="Datacenter → Backup job"
schedule    01:30
selection   all guests, VMs and LXCs (minus a few excluded ones)
mode        snapshot     # guest keeps running during the backup
fleecing    on           # buffer writes locally so a slow link doesn't stall the VM
compress    zstd
storage     pbs          # the remote PBS datastore
```

The data disks passed through to the VMs (the NAS's storage, the Docker host's media pool) are each marked **`backup=0`** in the VM's hardware settings, so the job only takes the OS disks. Mount points on LXC containers have the same `backup` flag. Those are what you need to get a VM booting again after a failure or a bad upgrade. The data on the other disks is covered by restic instead, file by file.

!!! tip
    Add PBS as storage with an [API token](https://pbs.proxmox.com/docs/user-management.html#api-tokens) that only has backup permissions on one datastore, rather than the root password.

## SnapRAID and mergerfs

The backup NAS runs [OpenMediaVault](https://www.openmediavault.org) with four large disks: three for data and one for parity.

- **[mergerfs](https://github.com/trapexit/mergerfs)** pools the three data disks into one folder (`/srv/mergerfs/<pool>`). Each file lives whole on one ordinary ext4 disk, so any disk can be pulled out and read on another machine.
- **[SnapRAID](https://www.snapraid.it)** computes parity across them on a schedule. If one disk dies, its contents can be rebuilt from the others plus parity.

```text title="/etc/snapraid.conf (shape)"
parity  /srv/dev-disk-by-uuid-<parity>/snapraid.parity
content /srv/dev-disk-by-uuid-<data1>/snapraid.content
content /srv/dev-disk-by-uuid-<data2>/snapraid.content
data d1 /srv/dev-disk-by-uuid-<data1>
data d2 /srv/dev-disk-by-uuid-<data2>
data d3 /srv/dev-disk-by-uuid-<data3>
exclude lost+found/
exclude /tmp/
```

OMV's SnapRAID plugin runs a nightly script that does `diff`, then `sync`, then a `scrub` once a week that re-reads everything and checks it against parity, and emails the result. It can also refuse to sync when a run would delete or change more files than a threshold (*delete/update threshold* in the plugin, where `0` turns the check off). Setting it means a mass deletion or ransomware run doesn't get written into the parity, so the files can still be recovered from it.

Why SnapRAID rather than RAID or ZFS here:

- **Mixed disk sizes are fine.** The parity disk just has to be as big as the largest data disk.
- **Only the disk being read spins up.** Good for a box that mostly sits idle.
- **No pool to lose.** Lose more disks than you have parity and you lose only *those* disks' files, not everything.

The catch is that parity is only as fresh as the last `sync`. A disk that dies mid-afternoon loses that day's new files. That suits a backup target or a media library (written once, read many times), not a busy database.

!!! note
    The main site's own media pool is ten btrfs disks under mergerfs with **no** parity. It's all replaceable, so a dead disk means re-downloading, not losing anything. Decide per dataset whether it's worth protecting.

## Testing restores

A backup you've never restored from is a hope, not a backup. Every so often:

- restore one service's folder from restic into `/tmp` and check it opens;
- load last night's database dump into a throwaway container;
- restore a VM or LXC from PBS under a new ID, boot it with networking off, and then delete it.
