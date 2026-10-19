# Samba (SMB shares)

The NAS shares its folders over **SMB**, the protocol behind Windows file sharing, using [Samba](https://www.samba.org). macOS, Windows and Linux all speak it natively. It's how other machines *use* the files without keeping their own copy: the Docker host mounts the photo library for Immich, and a laptop can open the shared library in Finder.

On the NAS it's set up through [OpenMediaVault](https://www.openmediavault.org)'s web UI (*Services → SMB/CIFS*), which writes `/etc/samba/smb.conf`. The settings below are what's worth changing from the defaults.

## Shares

One share per top-level folder (photos, personal files, the family library, a temporary scratch area), each with its own list of allowed users:

```ini title="smb.conf (one share)"
[gallery]
path = /srv/mergerfs/<pool>/gallery/
valid users = "<me>","<family member>"
read only = no
browseable = yes
guest ok = no
inherit acls = yes
inherit permissions = yes
```

Users are local accounts on the NAS with a separate Samba password (`smbpasswd -a <user>`, or *Users* in OMV).

## Making Macs happy

macOS works best with Samba's `fruit` module, which stores Finder metadata (tags, resource forks) the way macOS expects and makes browsing large folders much faster:

```ini title="smb.conf [global] (excerpt)"
vfs objects = catia fruit streams_xattr
fruit:aapl = yes
fruit:metadata = stream
fruit:encoding = native
fruit:copyfile = yes          # server-side copy: Finder copies within the NAS don't go over the network
fruit:zero_file_id = yes
server min protocol = SMB2_02
disable netbios = yes
```

With `fruit:time machine = yes` on a share, Macs can use that share as a Time Machine destination.

## Recycle bin

Samba's `recycle` module turns deletes over SMB into moves, so a file deleted by mistake from Finder or by an app isn't gone straight away:

```ini title="smb.conf (per share)"
vfs objects = catia fruit streams_xattr recycle
recycle:repository = .recycle/%U     # one bin per user
recycle:keeptree = yes                # keep the folder structure
recycle:versions = yes                # don't overwrite an earlier deleted copy
recycle:exclude = *.tmp, *.temp, ~$*
```

OMV can empty each share's `.recycle` after a set number of days. On plain Samba, a nightly `find .recycle -mtime +30 -delete` does the same. Leave `.recycle` out of backups.

## Mounting on Linux

The Docker host mounts the shares at boot via `/etc/fstab`:

```text title="/etc/fstab"
//<nas>/gallery   /mnt/nas-gallery   cifs   vers=3.0,credentials=/root/.smbcredentials,uid=1000,gid=1000,_netdev,nofail   0 0
```

```text title="/root/.smbcredentials (chmod 600)"
username=<user>
password=<password>
```

- **`credentials=`** keeps the password out of `fstab`, which any user can read.
- **`uid`/`gid`** make the files belong to the user the containers run as.
- **`_netdev,nofail`** wait for the network and don't hang the boot if the NAS is down.

Install `cifs-utils`, then `sudo mount -a` to test.

## Mounting on macOS

In Finder press `⌘K` and connect to `smb://<nas>/<share>`. Ticking *Remember this password* stores it in the keychain. To mount it at every login, drag the mounted share into *System Settings → General → Login Items*.

!!! note "Signing and encryption"
    My NAS has SMB signing and encryption turned off, for speed. That's only reasonable because the shares are reachable solely on the home LAN and over [Tailscale](../homelab/index.md#tailscale), which already encrypts everything. Never expose SMB (port 445) to the internet.
