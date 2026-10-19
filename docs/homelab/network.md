# Network

Three locations, all on one [Tailscale](index.md#tailscale) tailnet:

- **Main site:** the [Proxmox](proxmox.md) host with the NAS, the Docker host and the workstation, plus the UniFi controller.
- **Second site:** an OPNsense router, the backup NAS and Proxmox Backup Server.
- **Studio:** a GL.iNet travel router.

## OPNsense

[OPNsense](https://opnsense.org) is an open-source firewall and router based on FreeBSD. Most extra features come as plugins (*System → Firmware → Plugins*). The ones I use:

| Plugin | Why |
|---|---|
| `os-tailscale` | Puts the router on the tailnet as an **exit node**, so devices away from home can route their traffic through it. Optionally it's also a **subnet router**, making the whole LAN reachable remotely. |
| `os-adguardhome-maxit` | AdGuard Home as the network's DNS server (see below). Comes from the [mimugmail community repo](https://www.routerperformance.net/opnsense-repo/). |
| `os-crowdsec` | Same idea as on the [Docker host](docker.md#security-layers): blocks IPs with a bad reputation at the firewall. |
| `os-acme-client` | Let's Encrypt certificate for the web UI. |
| `os-mdns-repeater`, `os-udpbroadcastrelay` | Let discovery traffic (AirPlay, Chromecast, printers, game streaming) cross between VLANs. |
| `os-wol` | Wake-on-LAN from the web UI, for machines that sleep. |

## DNS: AdGuard Home

[AdGuard Home](https://adguard.com/en/adguard-home/overview.html) is a DNS server that blocks ads and trackers for every device on the network, phones and TVs included, without installing anything on them.

- It listens on port 53 on the router, so DHCP clients use it without any changes.
- The usual setup moves OPNsense's own resolver (Unbound) to another port and makes it AdGuard's upstream. Alternatively, AdGuard can use encrypted upstreams (DNS-over-HTTPS) directly.
- **Local names** are added as DNS rewrites, such as `*.<internal>.<domain>` → the Docker host. Internal services then resolve to a LAN IP and never leave the network. See [internal-only names](docker.md#security-layers).

Each site has its own AdGuard instance, so DNS keeps working if the tunnel between sites is down. [adguardhome-sync](https://github.com/bakito/adguardhome-sync) runs in Docker and copies the settings from one origin instance to the replicas every two hours, so blocklists, rewrites and client settings only need editing in one place:

```yaml title="compose excerpt"
  adguardhome-sync:
    image: ghcr.io/bakito/adguardhome-sync
    command: run
    environment:
      ORIGIN_URL: "http://<origin-adguard>:3000"
      ORIGIN_USERNAME: $ADGUARD_USER
      ORIGIN_PASSWORD: $ADGUARD_PASSWORD
      REPLICA1_URL: "http://<replica-adguard>"
      REPLICA1_USERNAME: $ADGUARD_USER
      REPLICA1_PASSWORD: $ADGUARD_PASSWORD
      CRON: "0 */2 * * *"
      RUN_ON_START: "true"
      FEATURES_DNS_SERVER_CONFIG: "false"   # each site keeps its own upstreams/listen settings
    restart: unless-stopped
```

!!! tip
    Give the router a second DNS server for DHCP clients only if it's another AdGuard instance. If the second one is a public resolver such as `1.1.1.1`, clients use it at random and ads leak through.

## Wi-Fi: UniFi

The UniFi access points are managed by the UniFi Network controller, which runs in a small **LXC container** on the Proxmox host (2 cores, 2 GB RAM) rather than on dedicated hardware. The access points keep working if the controller is off. It's only needed for changing settings and for stats.

## Travel router: GL.iNet

A [GL.iNet](https://www.gl-inet.com) travel router at the studio. It's small enough to take anywhere. It runs OpenWrt underneath, and GL.iNet's firmware has Tailscale built in (*Applications → Tailscale*). With Tailscale turned on, the router joins the tailnet, which gives you:

- every device behind it can reach home through the tailnet, including devices that can't run Tailscale themselves;
- it can use a home router as its **exit node**, so all its traffic leaves from home;
- when travelling, it can connect to hotel Wi-Fi (captive portal included) and re-broadcast it as your own network, so all your devices connect automatically and the hotel only sees one device.
