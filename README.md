# self-hosted-stack

Docker Compose stacks for a home server running Debian 13. Each subdirectory is a self-contained stack managed with `docker compose`.

## Stack index

| # | Stack | Services | Doc |
|---|---|---|---|
| 1 | `proxy/` | Nginx Proxy Manager, MariaDB | [01-proxy.md](docs/01-proxy.md) |
| 2 | `vaultwarden/` | Vaultwarden | [02-vaultwarden.md](docs/02-vaultwarden.md) |
| 3 | `arr/` | Prowlarr, FlareSolverr, qBittorrent, Radarr, Sonarr, Lidarr, Whisparr, Bazarr, Decluttarr, Maintainerr, Tdarr, Navidrome, Feishin | [03-arr.md](docs/03-arr.md) |
| 4 | `media/` | Jellyfin, Jellyseerr | [04-media.md](docs/04-media.md) |
| 5 | `searxng/` | SearXNG, Caddy, Valkey | [05-searxng.md](docs/05-searxng.md) |
| 6 | `audiobookshelf/` | Audiobookshelf | [06-audiobookshelf.md](docs/06-audiobookshelf.md) |
| 7 | `monitor/` | Uptime Kuma, MariaDB | [07-monitor.md](docs/07-monitor.md) |
| 8 | `immich-ml/` | Immich ML sidecar | [08-immich-ml.md](docs/08-immich-ml.md) |
| 9 | `watchtower/` | Watchtower | [09-watchtower.md](docs/09-watchtower.md) |

Deploy in the order shown — each stack depends on networks or services created by the ones above it.

## Prerequisites

**Start here:** [docs/00-prerequisites.md](docs/00-prerequisites.md)

Covers Docker installation, shared network creation, directory layout, and the shared `.env` file that all stacks reference.

## Quick start

```bash
git clone https://github.com/StepanKomis/self-hosted-stack.git stacks
cd stacks
cp .env.example .env   # edit with your values
```

Then follow the docs in order, beginning with the proxy stack.

## Hardware

- Intel i3-12100 · 16 GB RAM · Debian 13
- Intel iGPU → Jellyfin hardware transcoding (VAAPI)
- AMD RX 580 → Tdarr hardware transcoding (VAAPI + ROCm)
- 2.7 TB XFS data disk at `/mnt/nas`
