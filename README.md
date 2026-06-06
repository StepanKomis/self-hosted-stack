# self-hosted-stack

Docker Compose stacks for **bigboi** — a home server running Debian 13.

Each subdirectory is an independent stack. Run them with `docker compose` from inside the directory, or deploy via Portainer Agent (`192.168.0.20:9001`).

---

## Stacks

| Directory | Services |
|---|---|
| `proxy/` | Nginx Proxy Manager + MariaDB |
| `arr/` | Prowlarr, qBittorrent, Radarr, Sonarr, Lidarr, Whisparr, Bazarr, Flaresolverr, Decluttarr, Tdarr, Navidrome, Feishin, Maintainerr |
| `media/` | Jellyfin, Jellyseerr |
| `searxng/` | SearXNG, Caddy, Valkey |
| `vaultwarden/` | Vaultwarden |
| `audiobookshelf/` | Audiobookshelf |
| `monitor/` | Uptime Kuma + MariaDB |
| `immich-ml/` | Immich machine-learning sidecar |
| `pubrewind/` | PubRewind server, admin console, Postgres |
| `watchtower/` | Watchtower (auto-updates, deploy last) |

## Quick start

```bash
# Copy and fill in the env file
cp .env.example .env   # or restore your existing .env

# Start a stack
cd proxy
docker compose up -d
```

Shared env vars live in `.env` at the repo root. Each stack's `docker-compose.yml` references it via `env_file: ../.env`.

## Full deployment guide

See **[DEPLOY.md](DEPLOY.md)** for:

- Fresh OS install checklist
- Directory structure and bind mounts
- Deployment order and per-stack notes
- AMD/Intel GPU transcoding setup
- Nginx Proxy Manager routes
- Volume backup paths

## Hardware

- Intel i3-12100 · 16 GB RAM · Debian 13
- Intel iGPU → Jellyfin (OpenCL)
- AMD RX 580 → Tdarr (VAAPI / ROCm)
- 2.7 TB XFS data disk at `/mnt/nas`
