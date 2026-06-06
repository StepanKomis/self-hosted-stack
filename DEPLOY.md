# bigboi — standalone Docker deployment guide

No Portainer required. Each stack is a subdirectory; run `docker compose` from inside it.

---

## Quick reference

```bash
# Start a stack
cd /home/dfs/stacks/<name>
docker compose up -d

# Stop
docker compose down

# Logs
docker compose logs -f [service]

# Restart one service
docker compose restart <service>
```

---

## Pre-flight (fresh OS install only)

```bash
# 1. Install packages
apt install -y docker.io docker-compose-plugin openvpn tailscale nfs-kernel-server

# 2. Create directory structure
mkdir -p /srv/media/{movies,tv,music,podcasts,xxx}
mkdir -p /srv/{downloads,audiobookshelf,uptime-kuma,tdarr/transform_cache}
mkdir -p /opt/{proxy,kuma,servarr/bazarr}

# 3. Mount NAS disk (add to /etc/fstab first — see CLAUDE.md §2)
mount -a

# 4. Create shared Docker networks (only needed once)
docker network create nginx
docker network create proxy_external
docker network create servarr
docker network create media
docker network create music

# 5. VPN
systemctl enable --now openvpn-client@czech_republic

# 6. Tailscale
tailscale up
```

---

## Stack deployment order

### 1. proxy (Nginx Proxy Manager)

```bash
cd /home/dfs/stacks/proxy
docker compose up -d
```

Admin UI at `:81`. Default creds: `admin@example.com` / `changeme`.  
Restore `/opt/proxy/` from backup to bring back all proxy rules and SSL certs.

---

### 2. arr

```bash
cd /home/dfs/stacks/arr
docker compose up -d
```

Services: prowlarr · qbittorrent · radarr · sonarr · lidarr · whisparr · bazarr · flaresolverr · decluttarr · tdarr · navidrome · feishin · maintainerr

**After first start:**
1. Get API keys from each app (Settings → General → API Key)
2. Fill them into `/home/dfs/stacks/.env` under `RADARR_API_KEY`, etc.
3. `docker compose up -d` again to apply (decluttarr needs the keys)
4. In Prowlarr, add indexers and connect to Radarr/Sonarr/Lidarr/Whisparr

---

### 3. media (Jellyfin + Jellyseerr)

```bash
cd /home/dfs/stacks/media
docker compose up -d
```

#### AMD RX 5xx hardware transcoding setup

The compose file already sets `LIBVA_DRIVER_NAME=radeonsi` and passes `/dev/dri`.  
**Do not** add `DOCKER_MODS=jellyfin-opencl-intel` — it is Intel-only and will segfault on Polaris GPUs.

In Jellyfin UI (Dashboard → Playback → Transcoding):

| Setting | Value |
|---|---|
| Hardware acceleration | Video Acceleration API (VAAPI) |
| VA API Device | `/dev/dri/renderD128` |
| Enable hardware decoding | H264 ✓, HEVC ✓, VP8 ✓, VP9 ✓, AV1 ✓ (check all available) |
| Prefer hardware encoding | ✓ |
| Allow encoding in HEVC format | ✓ |

Verify VAAPI is visible inside the container:
```bash
docker exec -it jellyfin vainfo 2>&1 | grep -E "va_openDriver|VAProfile"
```

You should see `VAProfileH264`, `VAProfileHEVC`, etc.  
If `vainfo` says `failed with error -1` the render device permissions are wrong — check that `/dev/dri/renderD128` is group `render` (GID 108) and the container user is in that group (linuxserver handles this automatically via PUID/PGID).

---

### 4. searxng

```bash
# Clone upstream (includes Caddyfile template)
git clone https://github.com/searxng/searxng-docker.git /srv/searxng
# Restore your /srv/searxng/searxng/ config from backup, then:
cd /home/dfs/stacks/searxng
docker compose up -d
```

Note: searxng port is `127.0.0.1:8888:8080` (was conflicting with qbittorrent on 8080).  
If Caddy handles it externally this inner port doesn't matter; adjust if needed.

---

### 5. vaultwarden

```bash
cd /home/dfs/stacks/vaultwarden
docker compose up -d
```

Uses the existing `vaultwarden-data` Docker volume (marked `external` in compose).

---

### 6. audiobookshelf

```bash
cd /home/dfs/stacks/audiobookshelf
docker compose up -d
```

Config lives in `/srv/audiobookshelf/` — restore from backup before starting.

---

### 7. monitor (Uptime Kuma)

```bash
cd /home/dfs/stacks/monitor
docker compose up -d
```

Data: `/srv/uptime-kuma/` (app) and `/opt/kuma/` (MariaDB) — restore both from backup.

---

### 8. immich-ml

```bash
cd /home/dfs/stacks/immich-ml
docker compose up -d
```

Standalone ML sidecar; the full Immich instance lives on another host and calls port 3003.

---

### 9. pubrewind

```bash
# Build images first
docker build -t pubrewind-server:local /home/dfs/PubRewind-server/
docker build -t pubrewind-admin:local   /home/dfs/PubRewind-Admin-Console/
docker build -t pubdb:local -f /home/dfs/PubRewind-server/Dockerfile.db /home/dfs/PubRewind-server/

cd /home/dfs/stacks/pubrewind
docker compose up -d
```

The DB volume `pubrewind_db_data` is marked `external` — it survives redeployment.  
If the DB is still crashing, check logs: `docker logs pubrewind-database-server`

---

### 10. watchtower (deploy last)

```bash
cd /home/dfs/stacks/watchtower
docker compose up -d
```

Runs nightly at 04:00 Prague time. Skips stopped containers.

---

## Volume preservation

When running `docker compose up` from a directory named `<stack>`, Docker names
volumes as `<stack>_<volume_name>`. The directories here are named to match the
project names used by Portainer, so existing data volumes are automatically
reattached:

| Directory | Project name | Example volume created |
|---|---|---|
| `stacks/arr/` | `arr` | `arr_radarr_data` |
| `stacks/media/` | `media` | `media_jellyfin_config` |
| `stacks/searxng/` | `searxng` | `searxng_valkey_data` |
| `stacks/immich-ml/` | `immich-ml` | `immich-ml_model_cache` |
| `stacks/pubrewind/` | `pubrewind` | uses external `pubrewind_db_data` |

`vaultwarden` uses `external: true` explicitly so the oddly-named `vaultwarden-data` volume is found correctly.

---

## Known issues at last scan (2026-05-16)

| Service | Problem | Fix |
|---|---|---|
| `searxng` | OOM-killed (exit 137) | `mem_limit: 512m` added |
| `jellyfin-two` | Segfault (exit 139) | Intel opencl mod on AMD GPU — use this repo's media stack instead |
| `tdarr` | Unhealthy | AMD Polaris needs `/dev/kfd` + `HSA_OVERRIDE_GFX_VERSION=8.0.3` — both set |
| `immich-ml` | Unhealthy | Check model cache volume and connectivity to remote Immich host |
| `pubrewind-database-server` | Exited (1) | Postgres crash — check `docker logs` before restart |
| `proxy_db` | Exited (255) | MariaDB for NPM — restore `/opt/proxy/mysql` from backup |

---

## Proxy host routes (NPM)

All `.local` domains need a DNS entry or `/etc/hosts` pointing to `192.168.0.20`.

| Domain | Target | Port |
|---|---|---|
| `media.komis.cz` / `media.local` | jellyfin | 8096 |
| `search.komis.cz` / `search.local` | searxng caddy | 80/443 |
| `seer.local` | jellyseerr | 5055 |
| `radarr.local` | radarr | 7878 |
| `sonarr.local` | sonarr | 8989 |
| `lidarr.local` | lidarr | 8686 |
| `prowlarr.local` | prowlarr | 9696 |
| `torrent.local` | qbittorrent | 8080 |
| `tdarr.local` | tdarr | 8265 |
| `whisparr.local` | whisparr | 6969 |
| `bazarr.local` | bazarr | 6767 |
| `navidrome.local` | navidrome | 4533 |
| `music.local` | feishin | 9180 |
| `podcasts.local` | audiobookshelf | 8090 |
| `kuma.local` | uptime-kuma | 3001 |
| `pubrewind.local` | pubrewind-server (host) | 8082 |
| `admin.pubrewind.local` | pubrewind-admin | 3000 |
