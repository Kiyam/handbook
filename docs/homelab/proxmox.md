# Proxmox

Everything at the main site runs on one [Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview) host: the NAS, the Docker host and the workstation are VMs, and small services are LXC containers. Backups of all of them are covered in [Files → Backups](../files/backups.md#proxmox-backups).

## VM or LXC?

| | VM | LXC container |
|---|---|---|
| Kernel | Its own | Shares the host's |
| Overhead | A few GB of RAM, full OS | Tiny: boots in seconds, ~100 MB of RAM |
| Hardware | Full PCIe passthrough (GPU, disk controllers) | Bind mounts and device nodes only |
| Docker inside | Just works | Possible (`nesting=1`) but fiddly |
| Use it for | Anything with passthrough hardware, Docker hosts, other OSes (Home Assistant OS) | Single small services: the UniFi controller, a local PBS |

My rule: **one service and no special hardware → LXC; everything else → VM.**

For LXCs, the [Proxmox VE Helper-Scripts](https://community-scripts.github.io/ProxmoxVE/) are a quick start. They're one-line installers that create a ready-made, unprivileged container for a given app. Read the script before piping it into a root shell.

## GPU passthrough

The Docker host VM gets the whole NVIDIA GPU, which is used by Immich, Frigate, Plex/Jellyfin transcoding, Whisper and Ollama. The steps on an AMD host:

1. **Turn on IOMMU** (also called AMD-Vi or VT-d) in the BIOS, then on the kernel command line. Proxmox with LVM boots via GRUB: edit `/etc/default/grub`, run `update-grub` and reboot.

    ```text title="/etc/default/grub"
    GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt amd_iommu=on"
    ```

    For Intel, use `intel_iommu=on`. `iommu=pt` only maps devices that are passed through, which is faster for the host.

2. **Load VFIO and keep the host's own drivers off the GPU:**

    ```text title="/etc/modules"
    vfio
    vfio_iommu_type1
    vfio_pci
    ```

    ```text title="/etc/modprobe.d/blacklist.conf"
    blacklist nouveau
    blacklist nvidia*
    blacklist nvidiafb
    ```

    Then `update-initramfs -u -k all` and reboot.

3. **Check the IOMMU groups.** A device can only be passed through together with everything else in its group:

    ```bash title="iommu.sh"
    #!/bin/bash
    for d in /sys/kernel/iommu_groups/*/devices/*; do
      n=${d#*/iommu_groups/*}; n=${n%%/*}
      printf 'IOMMU Group %s ' "$n"
      lspci -nns "${d##*/}"
    done
    ```

    The GPU and its audio function should be alone in a group. If they share it with other devices, `pcie_acs_override=downstream` on the kernel command line splits the groups. It's a workaround that weakens isolation between those devices, so only add it if you need to. My board needs it.

4. **Configure the VM.** Use machine type `q35`, OVMF (UEFI) BIOS and CPU type `host`. Then add the GPU under *Hardware → Add → PCI Device* with *All Functions* and *PCI-Express* ticked:

    ```text title="qm config <vmid> (excerpt)"
    machine: q35
    bios: ovmf
    cpu: host
    balloon: 0
    hostpci0: 0000:81:00,pcie=1
    ```

    Turn memory ballooning off (`balloon: 0`). A VM with a passed-through device has to keep all its RAM pinned anyway.

5. **Inside the VM**, install the normal NVIDIA driver and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) so Docker containers can use the GPU.

!!! tip "NVENC session limit"
    Consumer NVIDIA cards cap the number of simultaneous hardware video encodes. With Plex, Jellyfin and Frigate all encoding, you hit it quickly. [keylase/nvidia-patch](https://github.com/keylase/nvidia-patch) removes the limit. Re-run it after every driver update.

## Passing disks through

The NAS and the Docker host get whole physical disks. Pass them by their stable `/dev/disk/by-id/` name, never `/dev/sdX`, because those letters can change between boots:

```bash
ls -l /dev/disk/by-id/ | grep -v part
qm set <vmid> -scsi1 /dev/disk/by-id/ata-<model>_<serial>,backup=0,iothread=1
```

`backup=0` keeps the disk out of the nightly vzdump job, since its data is backed up file by file with restic instead (see [Backups](../files/backups.md#proxmox-backups)). Add `discard=on` for SSDs.

The alternative is passing through a whole SATA or HBA controller as a PCI device. The VM then sees the disks directly, including SMART data and spin-down. That's worth doing if a NAS VM needs proper disk health monitoring.

## Small things worth doing

- **`qemu-guest-agent`** in every VM (`apt install qemu-guest-agent`, then tick *QEMU Guest Agent* under *Options*). Proxmox can then shut the VM down cleanly, show its IP, and freeze its filesystems for consistent snapshot backups.
- **`iothread=1`** with the *VirtIO SCSI single* controller gives each disk its own I/O thread.
- **Start order** under *Options → Start/Shutdown order*: start the NAS first and the Docker host a minute later, so the SMB mounts are there when containers start.
