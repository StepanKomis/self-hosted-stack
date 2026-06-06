# Prerequisites

Before deploying any stack, complete this page once on a fresh Debian or Ubuntu server.

---

## 1. Install Docker

```bash
# Debian / Ubuntu
apt update
apt install -y ca-certificates curl gnupg

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verify:

```bash
docker compose version
```

---

## 2. Clone this repository

```bash
git clone https://github.com/StepanKomis/self-hosted-stack.git /home/dfs/stacks
cd /home/dfs/stacks
```

> All stacks reference `../.env`, so they must live in subdirectories of a common parent.

---

## 3. Configure the shared environment file

```bash
cp .env.example .env
```

Edit `.env` and fill in your values:

```env
PUID=1000          # id -u
PGID=1000          # id -g
TZ=Europe/Prague   # IANA timezone string

# Paths — adjust to your disk layout
DOWNLOADS_DIR=/mnt/nas/qbittorrent
MOVIE_DIR=/mnt/nas/Media/Films
TV_DIR=/mnt/nas/Media/Shows
MUSIC_DIR=/mnt/nas/Media/Music
MEDIA_DIR=/mnt/nas/Media

TRANSCODE_CACHE_DIR=/srv/tdarr/transform_cache
TORRENTING_PORT=6881
SERVER_URL=media.example.com   # your Jellyfin public URL

# Filled in after the arr stack's first boot (see 04-arr.md)
RADARR_API_KEY=
SONARR_API_KEY=
LIDARR_API_KEY=
WHISPARR_API_KEY=

# Optional music integrations
ND_LASTFM_ENABLED=false
ND_LASTFM_APIKEY=
ND_LASTFM_SECRET=
ND_SPOTIFY_ID=
ND_SPOTIFY_SECRET=
FEISHIN_SERVER_NAME=Navidrome
```

---

## 4. Create host directories

```bash
# Media and downloads
mkdir -p /mnt/nas/Media/{Films,Shows,Music,Audiobooks}
mkdir -p /mnt/nas/qbittorrent

# App config / data dirs
mkdir -p /srv/{audiobookshelf/config,audiobookshelf/metadata,uptime-kuma,tdarr/transform_cache}
mkdir -p /opt/{proxy/data,proxy/letsencrypt,proxy/mysql,kuma,servarr/bazarr}
```

If your media lives on a separate disk, mount it first and add the entry to `/etc/fstab`:

```
UUID=<your-disk-uuid>  /mnt/nas  xfs  defaults,nofail  0  2
```

---

## 5. Create shared Docker networks

These networks must exist before any stack that references them as `external`:

```bash
docker network create nginx
docker network create proxy_external
docker network create servarr
docker network create media
docker network create music
```

> Each stack that *creates* a network (e.g., `servarr` in the arr stack) will not fail if the network already exists.

---

## 6. Deployment order

Deploy stacks in this order to satisfy network and volume dependencies:

| # | Stack | Key dependency |
|---|---|---|
| 1 | [proxy](01-proxy.md) | creates `nginx` and `proxy_external` networks |
| 2 | [vaultwarden](02-vaultwarden.md) | needs `nginx` |
| 3 | [arr](03-arr.md) | creates `servarr`, `music` networks |
| 4 | [media](04-media.md) | needs `servarr`, `nginx` |
| 5 | [searxng](05-searxng.md) | standalone |
| 6 | [audiobookshelf](06-audiobookshelf.md) | needs `nginx` |
| 7 | [monitor](07-monitor.md) | needs `nginx` |
| 8 | [immich-ml](08-immich-ml.md) | standalone sidecar |
| 9 | [watchtower](09-watchtower.md) | always last |
