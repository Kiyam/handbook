# Homelab

Notes on my homelab setup.

## Tailscale

Link: [tailscale.com](https://tailscale.com)

All my machines are on one [Tailscale](https://tailscale.com) network (a "tailnet"). Every machine gets a stable private IP and a MagicDNS name, and they can reach each other from anywhere without opening ports on the router or setting up a VPN server.

In practice this means `~/.ssh/config` entries simply point at the Tailscale address, and SSH, [Herdr's remote machines](../terminal/tmux-herdr.md#remote-machines) and everything else work the same at home or away:

```text title="~/.ssh/config"
Host <computer_name>
    HostName <tailscale_ip_or_magicdns_name>
```
