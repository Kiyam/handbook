# Services, monitoring & backups

## What runs

Grouped roughly by the compose file they live in. Not exhaustive, just the ones I'd recommend.

| Area | Services |
|---|---|
| Dashboards | [Homepage](https://gethomepage.dev), [Glance](https://github.com/glanceapp/glance) |
| Photos | [Immich](https://immich.app), with its machine-learning container on the GPU (CUDA) for face and object search |
| Media | Plex and [Jellyfin](https://jellyfin.org), [Audiobookshelf](https://www.audiobookshelf.org), the \*arr stack ([Sonarr](https://sonarr.tv), [Radarr](https://radarr.video), [Prowlarr](https://prowlarr.com), [Bazarr](https://www.bazarr.media)), [Tautulli](https://tautulli.com) |
| Documents | [Paperless-ngx](https://docs.paperless-ngx.com) (OCR and tagging for scanned documents), [Stirling PDF](https://www.stirlingpdf.com) |
| AI | [Ollama](https://ollama.com) + [Open WebUI](https://openwebui.com), [faster-whisper](https://github.com/SYSTRAN/faster-whisper) for speech-to-text, [ComfyUI](https://github.com/comfyanonymous/ComfyUI) |
| Home | [Frigate](https://frigate.video) (camera NVR with object detection on the GPU), [ntfy](https://ntfy.sh) (push notifications from scripts and services) |
| Reading & bookmarks | [Miniflux](https://miniflux.app) (RSS), [Karakeep](https://karakeep.app) (bookmarks), [Calibre](https://calibre-ebook.com) |
| Code & sync | [Forgejo](https://forgejo.org) (git), [Syncthing](https://syncthing.net), [Vaultwarden](https://github.com/dani-garcia/vaultwarden) (Bitwarden-compatible passwords) |
| Search & tools | [SearXNG](https://docs.searxng.org) (private meta search), [n8n](https://n8n.io) (automation), [changedetection.io](https://changedetection.io) |

!!! tip "Sharing one GPU"
    Immich ML, Frigate, Whisper and Ollama all use the same consumer GPU. Each gets the GPU through Compose's `deploy.resources.reservations.devices` block with the NVIDIA container toolkit. That works fine as long as they don't all peak at once. Ollama is the one to watch, since a big model holds most of the VRAM until it unloads.

## Storage

The Docker host keeps each service's config and databases on its own disk, and mounts bulk data from the NAS over SMB (`cifs` in `/etc/fstab`): photos for Immich, media, and the shared family library. Several local disks are pooled into one mount with [mergerfs](https://github.com/trapexit/mergerfs), so services see one big directory without caring which disk a file is on.

## Monitoring

- **[Beszel](https://beszel.dev).** A lightweight agent on each server reports CPU, memory, disk, temperatures and per-container stats to one dashboard, with alerts.
- **[Uptime Kuma](https://uptime.kuma.pet).** Checks each service's URL every minute and sends a notification when something goes down.
- **Autoheal.** Containers with a `healthcheck:` that turn unhealthy are restarted automatically. Add a healthcheck to anything that tends to hang rather than crash.
- **Mailrise.** Turns email into push notifications, for software that can only send alerts by email.

## Backups

- **Databases are dumped nightly** to `db_backups/` as dated `.sql.gz` files. A dump is consistent even while the app runs, which a copy of the live database directory isn't.
- **[Backrest](https://github.com/garethgeorge/backrest)** (a web UI for [restic](https://restic.net)) then snapshots the whole compose repo, the service data and the dumps, deduplicated and encrypted, on a schedule with retention rules.
- **Snapshot the VM in Proxmox** before a big upgrade, so a bad update is a rollback rather than a rebuild.

!!! warning "Test restores"
    A backup you have never restored from is a hope, not a backup. Every so often, restore one service's data into a scratch directory and check it opens.
