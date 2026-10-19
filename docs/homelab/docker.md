# Docker stack

One VM runs about a hundred containers, all from a single Compose project in a git repo. Every web service sits behind [Traefik](https://traefik.io), which handles HTTPS and routes `<service>.<domain>` to the right container.

## Layout: one root file, many includes

```text
docker/
├── docker-compose.yml       # networks, secrets, core services, and the include list
├── .env                     # shared variables: DOCKERDIR, DOMAINNAME, PUID/PGID, TZ ...
├── compose/
│   ├── media-management.yml # one file per group of related services
│   ├── media-management.env
│   ├── monitoring.yml
│   ├── monitoring.env
│   └── ...
├── traefik2/
│   ├── rules/               # Traefik dynamic config (middlewares, TLS, extra routes)
│   └── acme/                # certificates
├── secrets/                 # one file per secret, readable by root only
└── <service>/               # each service's persistent data
```

The root file pulls in every group with Compose's [`include`](https://docs.docker.com/compose/how-tos/multiple-compose-files/include/):

```yaml title="docker-compose.yml (excerpt)"
include:
  - path: $DOCKERDIR/compose/monitoring.yml
    env_file: $DOCKERDIR/compose/monitoring.env
  - path: $DOCKERDIR/compose/media-management.yml
    env_file: $DOCKERDIR/compose/media-management.env
  # ...
```

It's still one project, so `docker compose up -d` brings up everything and `docker compose ps` shows everything.

!!! warning "Always run Compose from the root"
    Restart a single service by name, from the repo root:

    ```bash
    docker compose up -d <service>
    docker compose pull <service> && docker compose up -d <service>
    ```

    **Don't** use `docker compose -f compose/<file>.yml up -d`. An included file on its own can't see the networks, secrets or `.env` defined in the root file, and fails with:

    ```text
    refers to undefined network traefik_proxy: invalid compose project
    ```

!!! tip
    I keep a `CLAUDE.md` in this repo with exactly these commands and gotchas, so Claude Code can manage the stack. See [A `CLAUDE.md` per project](../claude/claude-code-setup.md#a-claudemd-per-project).

## Networks and static IPs

The root file defines a few bridge networks with fixed subnets:

```yaml title="docker-compose.yml (excerpt)"
networks:
  traefik_proxy:          # every service Traefik routes to
    name: traefik_proxy
    driver: bridge
    ipam:
      config:
        - subnet: 172.18.0.0/24
  socket_proxy:           # only Traefik and the tools that need the Docker API
    name: socket_proxy
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

Each service gets a static address on `traefik_proxy`, so IPs don't move around when containers are recreated, and firewall or allow-list rules can refer to them:

```yaml
    networks:
      traefik_proxy:
        ipv4_address: 172.18.0.202
```

Databases go on their own small networks shared only with the app that uses them, not on `traefik_proxy`.

## Secrets

Passwords and API tokens are [Compose secrets](https://docs.docker.com/compose/how-tos/use-secrets/), one file each in `secrets/` (owned by root, mode `600`). They are mounted into containers at `/run/secrets/<name>` and are never written into the compose files:

```yaml
secrets:
  cf_token:
    file: $SECRETSDIR/cf_token

services:
  traefik:
    secrets:
      - cf_token
    environment:
      - CF_DNS_API_TOKEN_FILE=/run/secrets/cf_token
```

Most images accept a `*_FILE` variant of their environment variables for exactly this.

!!! danger "Keep secrets out of the repo"
    The compose files live in git, so anything pasted directly into `environment:` ends up in the history. Put it in a secret, or at least in a git-ignored `.env`.

## Traefik

### Docker provider through a socket proxy

Traefik reads container labels to build routes, so it needs the Docker API. Rather than mounting `/var/run/docker.sock` into an internet-facing container, a [socket proxy](https://github.com/Tecnativa/docker-socket-proxy) exposes a read-only subset of the API on the internal `socket_proxy` network, and Traefik talks to that:

```yaml
    command:
      - --providers.docker=true
      - --providers.docker.endpoint=tcp://socket-proxy:2375
      - --providers.docker.exposedByDefault=false
      - --providers.docker.network=traefik_proxy
      - --providers.docker.defaultrule=Host(`{{ index .Labels "com.docker.compose.service" }}.$DOMAINNAME`)
```

With `exposedByDefault=false`, nothing is published unless it has `traefik.enable=true`. The `defaultrule` means a service called `ntfy` is served at `ntfy.<domain>` without writing a `Host()` rule by hand.

### TLS

Certificates are a wildcard `*.<domain>` from Let's Encrypt via the **DNS challenge** (Cloudflare API token as a secret). Nothing has to be reachable from the internet to get a certificate, so internal-only services get real HTTPS too. The resolver and TLS options are set once on the `https` entrypoint, so individual services don't repeat them.

### Middleware chains

Middlewares are defined once in `traefik2/rules/middlewares.yml`, then grouped into named chains in `middleware-chains.yml`:

```yaml title="traefik2/rules/middleware-chains.yml (excerpt)"
http:
  middlewares:
    chain-no-auth:
      chain:
        middlewares:
          - middlewares-traefik-bouncer   # CrowdSec
          - middlewares-rate-limit
          - middlewares-https-redirectscheme
          - middlewares-secure-headers
    chain-tinyauth:
      chain:
        middlewares:
          - middlewares-traefik-bouncer
          - middlewares-rate-limit
          - middlewares-https-redirectscheme
          - middlewares-secure-headers
          - middlewares-tinyauth@docker    # login in front of the app
          - middlewares-compress
```

A service then only picks a chain:

```yaml title="compose/<group>.yml (excerpt)"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp-rtr.entrypoints=https"
      - "traefik.http.routers.myapp-rtr.middlewares=chain-tinyauth@file"
      - "traefik.http.services.myapp-svc.loadbalancer.server.port=8080"
```

Apps with their own good login use `chain-no-auth`. Apps with weak or no auth get `chain-tinyauth`, which puts [tinyauth](https://tinyauth.app) (single sign-on with OAuth) in front.

### Routes to things outside Docker

The file provider also holds `app-*.yml` routes for things that aren't containers on this host: the NAS web UI, the hypervisor, and services on other machines reached over [Tailscale](index.md#tailscale). They get the same certificates and middleware chains as everything else.

## Security layers

- **CrowdSec.** It reads Traefik's access logs and blocks IPs that behave badly (scanners, brute force), using its community blocklists too. It enforces this in two places: a Traefik plugin bouncer (the `middlewares-traefik-bouncer` at the start of every chain) and a firewall bouncer on the host.
- **Default deny.** `exposedByDefault=false`, plus an auth middleware for anything without its own login.
- **Internal-only names enforced in Traefik, not just DNS.** Internal services use their own subdomain, `*.<internal>.<domain>`, which has no public DNS record. That alone isn't enough: port 443 is forwarded to Traefik, so anyone who sends the right `Host` header to the public IP still reaches them. A high-priority router catches every internal name and returns 403 unless the request comes from the LAN, Tailscale (`100.64.0.0/10`) or Docker's own networks:

    ```yaml title="traefik2/rules/internal-lan-only.yml (excerpt)"
    http:
      routers:
        internal-block-external:
          entryPoints: [https]
          rule: "HostRegexp(`(?i)^.+\\.<internal>\\.<domain>$`) && !(ClientIP(`192.168.1.0/24`) || ClientIP(`100.64.0.0/10`) || ClientIP(`172.16.0.0/12`))"
          priority: 100000          # outranks every other router
          service: noop@internal
          middlewares: [internal-deny-all]
    ```
- **Socket proxy** instead of the raw Docker socket, as above.
