# 04 — Media (Jellyfin + Jellyseerr)

The media server stack. Jellyfin streams video and audio to any device; Jellyseerr is the request front-end that lets users discover content and submit requests that feed into Radarr/Sonarr automatically.

## Services

| Container | Image | Role | Port |
|---|---|---|---|
| `jellyfin` | linuxserver/jellyfin | Media streaming server | 8096 |
| `jellyseerr` | fallenbagel/jellyseerr | Media request portal | 5055 |

## Dependencies

- `servarr` Docker network — created by the [arr stack](03-arr.md)
- `nginx` Docker network — created by the [proxy stack](01-proxy.md)
- `MEDIA_DIR` on the host, readable by `PUID:PGID`

## Networks

| Network | Type | Role |
|---|---|---|
| `media` | internal bridge | Jellyfin ↔ Jellyseerr |
| `servarr` | external | Jellyseerr → Radarr / Sonarr |
| `nginx` | external | Proxy access |

## `.env` variables used

```env
PUID=1000
PGID=1000
TZ=Europe/Prague
SERVER_URL=media.example.com   # public Jellyfin URL for client discovery
MEDIA_DIR=/mnt/nas/Media
```

## Compose file

```yaml
volumes:
  jellyfin_config:
  jellyfin_cache:
  jellyseerr_data:

networks:
  media:
    name: media
    driver: bridge
    internal: true
  servarr:
    name: servarr
    external: true
  nginx:
    name: nginx
    external: true

x-logging: &default-logging
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"

services:

  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    logging: *default-logging
    env_file: ../.env
    environment:
      PUID: ${PUID}
      PGID: ${PGID}
      TZ: ${TZ}
      JELLYFIN_PublishedServerUrl: ${SERVER_URL}
      LIBVA_DRIVER_NAME: radeonsi
    volumes:
      - jellyfin_config:/config
      - jellyfin_cache:/cache
      - ${MEDIA_DIR}:/media:ro
    devices:
      - /dev/dri:/dev/dri
    ports:
      - "8096:8096"
      - "8920:8920"
    networks:
      - media
      - nginx
    mem_limit: 4g
    cpus: 8.0
    healthcheck:
      test: ["CMD-SHELL", "curl -fsSL http://localhost:8096/health -o /dev/null || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    restart: unless-stopped
    logging: *default-logging
    init: true
    env_file: ../.env
    environment:
      TZ: ${TZ}
      LOG_LEVEL: info
      PORT: 5055
    volumes:
      - jellyseerr_data:/app/config
    ports:
      - "5055:5055"
    networks:
      - media
      - servarr
      - nginx
    mem_limit: 256m
    cpus: 2.0
    depends_on:
      jellyfin:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -fsSL http://localhost:5055/api/v1/status -o /dev/null || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s
```

## Deploy

```bash
cd /home/dfs/stacks/media
docker compose up -d
```

## Hardware transcoding — AMD RX 5xx (Polaris)

The compose file sets `LIBVA_DRIVER_NAME=radeonsi` and passes `/dev/dri` into Jellyfin. This is the correct setup for AMD RX 5xx (Polaris) and RX 6xx series cards using the open-source Mesa driver.

> Do **not** add `DOCKER_MODS=jellyfin-opencl-intel` — that mod is Intel-only and causes a segfault on AMD hardware.

After first boot, configure in Jellyfin UI:

**Dashboard → Playback → Transcoding**

| Setting | Value |
|---|---|
| Hardware acceleration | Video Acceleration API (VAAPI) |
| VA API Device | `/dev/dri/renderD128` |
| H264 / HEVC / VP8 / VP9 / AV1 | Enable all available |
| Prefer hardware encoding | ✓ |
| Allow encoding in HEVC format | ✓ |

Verify VAAPI is functional inside the container:

```bash
docker exec -it jellyfin vainfo 2>&1 | grep -E "va_openDriver|VAProfile"
```

You should see `VAProfileH264`, `VAProfileHEVC`, etc. If `vainfo` reports `failed with error -1`, check that `/dev/dri/renderD128` belongs to group `render` (GID 108) — linuxserver images handle group membership automatically via PUID/PGID.

**Intel iGPU:** replace `LIBVA_DRIVER_NAME: radeonsi` with `LIBVA_DRIVER_NAME: i965` (or remove it to let the driver auto-detect).

**No GPU:** remove the `devices` block; Jellyfin falls back to software transcoding.

## Jellyfin first-run

1. Open `http://<server-ip>:8096`
2. Follow the setup wizard — create your admin account and add your media libraries:
   - Movies → `/media/Films`
   - TV Shows → `/media/Shows`
   - Music → `/media/Music`
3. Set `SERVER_URL` in `.env` to your public domain and run `docker compose up -d` again so client discovery advertises the correct URL

## Jellyseerr first-run

1. Open `http://<server-ip>:5055`
2. Sign in with your Jellyfin account
3. Connect to Radarr: hostname `radarr`, port `7878`, API key from `.env`
4. Connect to Sonarr: hostname `sonarr`, port `8989`, API key from `.env`

## Nginx Proxy Manager routes

| Domain | Forward to | Port |
|---|---|---|
| `media.yourdomain.com` | `jellyfin` | `8096` |
| `seer.yourdomain.com` | `jellyseerr` | `5055` |

## Ports summary

| Port | Service |
|---|---|
| 8096 | Jellyfin HTTP |
| 8920 | Jellyfin HTTPS (optional) |
| 5055 | Jellyseerr |
