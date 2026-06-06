# 06 — Audiobookshelf

A self-hosted audiobook and podcast server with a clean web UI and mobile apps (iOS and Android) for offline listening and progress sync.

## Services

| Container | Image | Role | Port |
|---|---|---|---|
| `audiobookshelf` | `advplyr/audiobookshelf` | Audiobook + podcast server | 8090 |

## Dependencies

- `nginx` Docker network — created by the [proxy stack](01-proxy.md)

## Host directories

The container uses bind mounts (not named volumes) so data survives a `docker compose down -v`:

| Host path | Container path | Content |
|---|---|---|
| `/srv/audiobookshelf/config` | `/config` | App config, users, settings |
| `/srv/audiobookshelf/metadata` | `/metadata` | Cover art, chapter files |
| `/srv/media/podcasts` | `/podcasts` | Podcast RSS/audio files |
| `/mnt/nas/Media/Audiobooks` | `/audiobooks` | Audiobook library |

Create them before deploying:

```bash
mkdir -p /srv/audiobookshelf/{config,metadata}
mkdir -p /srv/media/podcasts
mkdir -p /mnt/nas/Media/Audiobooks
```

## Compose file

```yaml
networks:
  nginx:
    name: nginx
    external: true

services:
  audiobookshelf:
    image: ghcr.io/advplyr/audiobookshelf:latest
    container_name: audiobookshelf
    restart: unless-stopped
    ports:
      - "8090:80"
    volumes:
      - /srv/audiobookshelf/config:/config
      - /srv/audiobookshelf/metadata:/metadata
      - /srv/media/podcasts:/podcasts
      - /mnt/nas/Media/Audiobooks:/audiobooks
    networks:
      - nginx
```

## Deploy

```bash
cd /home/dfs/stacks/audiobookshelf
docker compose up -d
```

## First-run

1. Open `http://<server-ip>:8090`
2. Create the root admin account on first visit
3. Add libraries:
   - **Audiobooks** → `/audiobooks`
   - **Podcasts** → `/podcasts`
4. Trigger a scan — Audiobookshelf reads ID3 tags and folder structure for metadata

## Nginx Proxy Manager setup

| Field | Value |
|---|---|
| Domain | `podcasts.yourdomain.com` |
| Forward Hostname | `audiobookshelf` |
| Forward Port | `80` |
| SSL | Let's Encrypt ✓ |
| Websockets | Enable (required for real-time progress sync) |

## Mobile apps

The official Audiobookshelf apps connect to your server URL. They support offline download and cross-device progress sync via the server. No account with Audiobookshelf Inc. is needed — everything is your own instance.

## Backup

```bash
tar czf abs-backup-$(date +%F).tar.gz \
  /srv/audiobookshelf/config \
  /srv/audiobookshelf/metadata
```

Your audiobook files live on the NAS and do not need to be included in the backup — only config and metadata do.
