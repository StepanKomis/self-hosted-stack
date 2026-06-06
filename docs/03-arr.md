# 03 — Arr Stack

The media automation suite. This single stack handles indexing, downloading, library management, subtitle fetching, transcoding, music streaming, and queue maintenance.

## Services

| Container | Image | Role | Port |
|---|---|---|---|
| `prowlarr` | linuxserver/prowlarr | Indexer manager (feeds all *arrs) | 9696 |
| `flaresolverr` | flaresolverr/flaresolverr | Cloudflare bypass for indexers | 8191 |
| `qbittorrent` | linuxserver/qbittorrent | Torrent client | 8080 |
| `radarr` | linuxserver/radarr | Movie library manager | 7878 |
| `sonarr` | linuxserver/sonarr | TV library manager | 8989 |
| `lidarr` | linuxserver/lidarr | Music library manager | 8686 |
| `whisparr` | thespad/whisparr | Adult content library manager | 6969 |
| `bazarr` | linuxserver/bazarr | Subtitle manager | 6767 |
| `decluttarr` | manimatter/decluttarr | Queue cleaner (removes stalled/failed) | — |
| `maintainerr` | jorenn92/maintainerr | Library rule engine / auto-delete | 6246 |
| `tdarr` | haveagitgat/tdarr | Video transcoding server + worker | 8265 / 8266 |
| `navidrome` | deluan/navidrome | Self-hosted music streaming server | 4533 |
| `feishin` | jeffvli/feishin | Navidrome web frontend | 9180 |

## Networks created

| Network | Type | Used by |
|---|---|---|
| `servarr` | bridge | All arr services, Jellyseerr |
| `music` | bridge | Navidrome, Feishin |

## Networks required (external)

- `nginx` — created by the [proxy stack](01-proxy.md)

## `.env` variables used

```env
PUID=1000
PGID=1000
TZ=Europe/Prague

DOWNLOADS_DIR=/mnt/nas/qbittorrent
MOVIE_DIR=/mnt/nas/Media/Films
TV_DIR=/mnt/nas/Media/Shows
MUSIC_DIR=/mnt/nas/Media/Music
MEDIA_DIR=/mnt/nas/Media

TRANSCODE_CACHE_DIR=/srv/tdarr/transform_cache
TORRENTING_PORT=6881

# Filled in after first boot:
RADARR_API_KEY=
SONARR_API_KEY=
LIDARR_API_KEY=
WHISPARR_API_KEY=

# Optional:
ND_LASTFM_ENABLED=false
ND_LASTFM_APIKEY=
ND_LASTFM_SECRET=
ND_SPOTIFY_ID=
ND_SPOTIFY_SECRET=
FEISHIN_SERVER_NAME=Navidrome
```

## Host directories required

```bash
mkdir -p /opt/servarr/bazarr
mkdir -p /srv/tdarr/transform_cache
```

## Hardware note — AMD GPU (Tdarr)

Tdarr uses VAAPI + ROCm for hardware-accelerated transcoding on AMD cards. The compose file passes `/dev/dri` and `/dev/kfd` and sets `HSA_OVERRIDE_GFX_VERSION=8.0.3` for Polaris (RX 5xx) GPUs.

If you are using an **Intel** iGPU, remove the `/dev/kfd` device and the `ROC_ENABLE_PRE_VEGA` / `HSA_OVERRIDE_GFX_VERSION` env vars.

If you have **no GPU**, remove the `devices` block from the Tdarr service and use CPU transcoding.

## Compose file

```yaml
volumes:
  radarr_data:
  sonarr_data:
  lidarr_data:
  whisparr_data:
  prowlarr_data:
  qbittorrent_data:
  tdarr_server:
  tdarr_configs:
  tdarr_logs:
  navidrome_data:
  maintainerr_data:

networks:
  servarr:
    name: servarr
    driver: bridge
  nginx:
    name: nginx
    external: true
  music:
    name: music
    driver: bridge

x-logging: &default-logging
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "1"

x-arr-base: &arr-base
  restart: unless-stopped
  networks:
    - servarr
    - nginx
  logging: *default-logging

services:

  # ── Indexer ──────────────────────────────────────────────────────────────────

  prowlarr:
    <<: *arr-base
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    env_file: ../.env
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - prowlarr_data:/config
    ports:
      - "9696:9696"
    mem_limit: 512m
    cpus: 2
    healthcheck:
      test: curl -f http://localhost:9696/api/v1/system/status || exit 1
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s

  flaresolverr:
    <<: *arr-base
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    env_file: ../.env
    environment:
      - LOG_LEVEL=info
      - TZ=${TZ}
    ports:
      - "8191:8191"
    mem_limit: 512m
    cpus: 2
    healthcheck:
      test: curl -f http://localhost:8191 || exit 1
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ── Downloader ───────────────────────────────────────────────────────────────

  qbittorrent:
    <<: *arr-base
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    env_file: ../.env
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=8080
      - TORRENTING_PORT=${TORRENTING_PORT}
    volumes:
      - qbittorrent_data:/config
      - ${DOWNLOADS_DIR}:/downloads
    ports:
      - "8080:8080"
      - "${TORRENTING_PORT}:${TORRENTING_PORT}"
      - "${TORRENTING_PORT}:${TORRENTING_PORT}/udp"
    healthcheck:
      test: curl -f http://localhost:8080 || exit 1
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ── Library managers ─────────────────────────────────────────────────────────

  radarr:
    <<: *arr-base
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    env_file: ../.env
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - radarr_data:/config
      - ${MOVIE_DIR}:/movies
      - ${DOWNLOADS_DIR}:/downloads
    ports:
      - "7878:7878"
    mem_limit: 512m
    cpus: 2
    depends_on:
      qbittorrent:
        condition: service_healthy
      prowlarr:
        condition: service_healthy
    healthcheck:
      test: curl -f http://localhost:7878/api/v3/system/status || exit 1
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 40s

  sonarr:
    <<: *arr-base
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    env_file: ../.env
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - sonarr_data:/config
      - ${TV_DIR}:/tv
      - ${DOWNLOADS_DIR}:/downloads
    ports:
      - "8989:8989"
    mem_limit: 512m
    cpus: 2
    depends_on:
      qbittorrent:
        condition: service_healthy
      prowlarr:
        condition: service_healthy
    healthcheck:
      test: curl -f http://localhost:8989/api/v3/system/status || exit 1
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 40s

  lidarr:
    <<: *arr-base
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    env_file: ../.env
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - lidarr_data:/config
      - ${MUSIC_DIR}:/music
      - ${DOWNLOADS_DIR}:/downloads
    ports:
      - "8686:8686"
    mem_limit: 1g
    cpus: 4
    depends_on:
      qbittorrent:
        condition: service_healthy
      prowlarr:
        condition: service_healthy
    healthcheck:
      test: curl -f http://localhost:8686/api/v1/system/status || exit 1
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s

  whisparr:
    <<: *arr-base
    image: ghcr.io/thespad/whisparr:latest
    container_name: whisparr
    env_file: ../.env
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - UMASK=002
    volumes:
      - whisparr_data:/config
      - ${ADULT_DIR}:/adult
      - ${DOWNLOADS_DIR}:/downloads
    ports:
      - "6969:7878"
    mem_limit: 512m
    cpus: 2
    depends_on:
      qbittorrent:
        condition: service_healthy
      prowlarr:
        condition: service_healthy
    healthcheck:
      test: curl -f http://localhost:7878/api/v3/system/status || exit 1
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 40s

  bazarr:
    <<: *arr-base
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    env_file: ../.env
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - /opt/servarr/bazarr:/config
      - ${MOVIE_DIR}:/movies
      - ${TV_DIR}:/tv
    ports:
      - "6767:6767"
    mem_limit: 512m
    cpus: 2
    healthcheck:
      test: curl -f http://localhost:6767 || exit 1
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ── Queue maintenance ─────────────────────────────────────────────────────────

  decluttarr:
    <<: *arr-base
    image: ghcr.io/manimatter/decluttarr:latest
    container_name: decluttarr
    env_file: ../.env
    environment:
      - TZ=${TZ}
      - LOG_LEVEL=INFO
      - RADARR_URL=http://radarr:7878
      - RADARR_KEY=${RADARR_API_KEY}
      - SONARR_URL=http://sonarr:8989
      - SONARR_KEY=${SONARR_API_KEY}
      - LIDARR_URL=http://lidarr:8686
      - LIDARR_KEY=${LIDARR_API_KEY}
      - WHISPARR_URL=http://whisparr:7878
      - WHISPARR_KEY=${WHISPARR_API_KEY}
      - QBITTORRENT_URL=http://qbittorrent:8080
      - REMOVE_TIMER=10
      - REMOVE_FAILED=true
      - REMOVE_FAILED_IMPORTS=true
      - REMOVE_METADATA_MISSING=true
      - REMOVE_MISSING_FILES=true
      - REMOVE_ORPHANS=true
      - REMOVE_SLOW=false
      - REMOVE_STALLED=true
      - REMOVE_UNMONITORED=false
      - MIN_DOWNLOAD_SPEED=0
      - PERMITTED_ATTEMPTS=3
      - NO_INTERNET_RETRY_ATTEMPTS=3
      - FAILED_IMPORT_MESSAGE_PATTERNS=["Not a Custom Format upgrade", "Not an upgrade for existing"]
    mem_limit: 128m
    cpus: 1
    depends_on:
      radarr:
        condition: service_healthy
      sonarr:
        condition: service_healthy

  maintainerr:
    <<: *arr-base
    image: ghcr.io/jorenn92/maintainerr:latest
    container_name: maintainerr
    env_file: ../.env
    user: "${PUID}:${PGID}"
    environment:
      - TZ=${TZ}
    volumes:
      - maintainerr_data:/opt/data
    ports:
      - "6246:6246"
    mem_limit: 256m
    cpus: 1
    healthcheck:
      test: curl -f http://localhost:6246/api/health || exit 1
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ── Transcoding — AMD RX 580 VAAPI + ROCm ────────────────────────────────────

  tdarr:
    <<: *arr-base
    image: ghcr.io/haveagitgat/tdarr:latest
    container_name: tdarr
    env_file: ../.env
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - UMASK_SET=002
      - serverIP=0.0.0.0
      - serverPort=8266
      - webUIPort=8265
      - internalNode=true
      - inContainer=true
      - ffmpegVersion=6
      - nodeName=MainNode
      - LIBVA_DRIVER_NAME=radeonsi
      - ROC_ENABLE_PRE_VEGA=1
      - HSA_OVERRIDE_GFX_VERSION=8.0.3
    volumes:
      - tdarr_server:/app/server
      - tdarr_configs:/app/configs
      - tdarr_logs:/app/logs
      - ${MEDIA_DIR}:/media
      - ${TRANSCODE_CACHE_DIR}:/temp
    devices:
      - /dev/dri:/dev/dri
      - /dev/kfd:/dev/kfd
    ports:
      - "8265:8265"
      - "8266:8266"
    mem_limit: 4g
    cpus: 8
    healthcheck:
      test: curl -f http://localhost:8265 || exit 1
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ── Music streaming ───────────────────────────────────────────────────────────

  navidrome:
    image: deluan/navidrome:latest
    container_name: navidrome
    restart: unless-stopped
    logging: *default-logging
    env_file: ../.env
    user: "${PUID}:${PGID}"
    environment:
      - TZ=${TZ}
      - ND_SCANSCHEDULE=1h
      - ND_LOGLEVEL=info
      - ND_SESSIONTIMEOUT=24h
      - ND_ENABLETRANSCODINGCONFIG=true
      - ND_TRANSCODINGCACHESIZE=500MB
      - ND_IMAGECACHESIZE=100MB
      - ND_LASTFM_ENABLED=${ND_LASTFM_ENABLED:-false}
      - ND_LASTFM_APIKEY=${ND_LASTFM_APIKEY:-}
      - ND_LASTFM_SECRET=${ND_LASTFM_SECRET:-}
      - ND_SPOTIFY_ID=${ND_SPOTIFY_ID:-}
      - ND_SPOTIFY_SECRET=${ND_SPOTIFY_SECRET:-}
    volumes:
      - navidrome_data:/data
      - ${MUSIC_DIR}:/music:ro
    ports:
      - "4533:4533"
    networks:
      - servarr
      - nginx
      - music
    mem_limit: 512m
    cpus: 2
    healthcheck:
      test: curl -f http://localhost:4533/ping || exit 1
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s

  feishin:
    image: ghcr.io/jeffvli/feishin:latest
    container_name: feishin
    restart: unless-stopped
    logging: *default-logging
    env_file: ../.env
    environment:
      - SERVER_NAME=${FEISHIN_SERVER_NAME:-Navidrome}
      - SERVER_TYPE=navidrome
      - SERVER_URL=http://navidrome:4533
      - SERVER_LOCK=false
    ports:
      - "9180:9180"
    networks:
      - servarr
      - nginx
      - music
    mem_limit: 128m
    cpus: 1
    depends_on:
      navidrome:
        condition: service_healthy
    healthcheck:
      test: curl -f http://localhost:9180 || exit 1
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 20s
```

## Deploy

```bash
cd /home/dfs/stacks/arr
docker compose up -d
```

## Post-first-boot setup

Decluttarr needs API keys from each *arr app before it can function. After the first boot:

1. Open each app's web UI and navigate to **Settings → General → API Key**
2. Copy the key for each app into `.env`:

```bash
# In /home/dfs/stacks/.env
RADARR_API_KEY=xxxxxxxxxxxxxxxx
SONARR_API_KEY=xxxxxxxxxxxxxxxx
LIDARR_API_KEY=xxxxxxxxxxxxxxxx
WHISPARR_API_KEY=xxxxxxxxxxxxxxxx
```

3. Restart the stack to apply:

```bash
docker compose up -d
```

## Prowlarr — connect indexers to *arrs

1. In Prowlarr → **Settings → Apps**, add each app:
   - Use the container name as the hostname (e.g. `http://radarr:7878`)
   - Paste the API key
2. Add your indexers under **Indexers**
3. Prowlarr pushes indexer config to Radarr/Sonarr/Lidarr/Whisparr automatically

## Flaresolverr — Cloudflare bypass

In Prowlarr → **Settings → Indexers**, set the FlareSolverr URL for indexers that require it:

```
http://flaresolverr:8191
```

## Bazarr — subtitle setup

1. Open Bazarr at `:6767`
2. **Settings → Radarr** — hostname: `radarr`, port: `7878`, API key from `.env`
3. **Settings → Sonarr** — same pattern
4. Enable subtitle providers under **Settings → Providers**

## Tdarr — GPU transcoding (AMD Polaris)

Tdarr uses VAAPI for decoding and ROCm for processing. On AMD RX 5xx (Polaris) GPUs, the GFX version must be overridden because the card predates the ROCm version check:

```yaml
- HSA_OVERRIDE_GFX_VERSION=8.0.3   # RX 580 = GFX 8.0.3
- ROC_ENABLE_PRE_VEGA=1
```

After starting, open Tdarr at `:8265` and configure a library pointing to `/media`.

## Ports summary

| Port | Service |
|---|---|
| 8080 | qBittorrent WebUI |
| 9696 | Prowlarr |
| 8191 | FlareSolverr |
| 7878 | Radarr |
| 8989 | Sonarr |
| 8686 | Lidarr |
| 6969 | Whisparr |
| 6767 | Bazarr |
| 6246 | Maintainerr |
| 8265 | Tdarr WebUI |
| 8266 | Tdarr server API |
| 4533 | Navidrome |
| 9180 | Feishin |
