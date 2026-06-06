# 07 — Monitor (Uptime Kuma)

A self-hosted uptime and status monitoring tool. Tracks HTTP endpoints, TCP ports, DNS, and more, with notification integrations for Telegram, Discord, Slack, email, and others.

## Services

| Container | Image | Role | Port |
|---|---|---|---|
| `uptime-kuma` | `louislam/uptime-kuma` | Monitoring UI + engine | 3001 |
| `kuma-db` | `mariadb:11` | Persistent monitor database | — |

## Networks

| Network | Type | Role |
|---|---|---|
| `nginx` | external | Proxy access to web UI |
| `monitor_internal` | internal | Kuma ↔ MariaDB only |

## Host directories

```bash
mkdir -p /srv/uptime-kuma   # app data
mkdir -p /opt/kuma           # MariaDB data
```

## Compose file

```yaml
networks:
  nginx:
    name: nginx
    external: true
  monitor_internal:
    name: monitor_internal
    internal: true

services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - /srv/uptime-kuma:/app/data
    networks:
      - nginx
      - monitor_internal
    ports:
      - "3001:3001"

  kuma-db:
    image: mariadb:11
    container_name: kuma-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: kuma
      MYSQL_DATABASE: kuma
      MYSQL_USER: kuma
      MYSQL_PASSWORD: kuma
    volumes:
      - /opt/kuma:/var/lib/mysql
    networks:
      - monitor_internal
```

> Change the MariaDB passwords before deploying if this server has any network exposure.

## Deploy

```bash
cd /home/dfs/stacks/monitor
docker compose up -d
```

## First-run

1. Open `http://<server-ip>:3001`
2. Create the admin account on first visit
3. Add monitors — examples:

| Name | Type | URL / Host |
|---|---|---|
| Nginx Proxy Manager | HTTP | `http://proxy:81` |
| Jellyfin | HTTP | `http://jellyfin:8096/health` |
| Vaultwarden | HTTP | `https://vault.yourdomain.com` |
| Navidrome | HTTP | `http://navidrome:4533/ping` |
| qBittorrent | HTTP | `http://qbittorrent:8080` |

Because `uptime-kuma` is on the `nginx` network (same bridge as the other stacks), you can use container names directly as hostnames.

## Nginx Proxy Manager setup

| Field | Value |
|---|---|
| Domain | `kuma.yourdomain.com` |
| Forward Hostname | `uptime-kuma` |
| Forward Port | `3001` |
| SSL | Let's Encrypt ✓ |
| Websockets | Enable (required — Kuma uses WebSockets for the live dashboard) |

## Notifications

Configure in **Settings → Notifications**. Telegram and Discord are the easiest — both require only a bot token or webhook URL.

## Backup

All state lives in `/srv/uptime-kuma` (SQLite database + config). Back it up while Kuma is running — SQLite supports hot reads.

```bash
tar czf kuma-backup-$(date +%F).tar.gz /srv/uptime-kuma
```
