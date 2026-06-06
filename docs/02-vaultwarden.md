# 02 — Vaultwarden

A lightweight, self-hosted Bitwarden-compatible password manager. All Bitwarden clients (browser extension, desktop, mobile) work against it without modification.

> **HTTPS is required.** Bitwarden clients refuse to connect over plain HTTP. Set up a proxy host in NPM with a valid SSL certificate before attempting to use any client.

## Services

| Container | Image | Role |
|---|---|---|
| `vaultwarden` | `vaultwarden/server` | Password manager server |

## Dependencies

- `nginx` Docker network (created by the [proxy stack](01-proxy.md))
- A Docker volume named exactly `vaultwarden-data` (note the hyphen — Docker does not auto-create this because the compose file marks it `external`)

## Create the volume before first deploy

```bash
docker volume create vaultwarden-data
```

## Compose file

```yaml
volumes:
  vaultwarden_data:
    external: true
    name: vaultwarden-data

networks:
  nginx:
    name: nginx
    external: true

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    ports:
      - "90:80"
    volumes:
      - vaultwarden_data:/data
    networks:
      - nginx
```

## Deploy

```bash
cd /home/dfs/stacks/vaultwarden
docker compose up -d
```

## Nginx Proxy Manager setup

| Field | Value |
|---|---|
| Domain | `vault.yourdomain.com` |
| Forward Hostname | `vaultwarden` |
| Forward Port | `80` |
| SSL | Request Let's Encrypt certificate ✓ |
| Websockets | Enable (required for live sync) |

## First-run

1. Navigate to `https://vault.yourdomain.com`
2. Click **Create Account** to register the admin account
3. (Optional) Disable public registration by setting `SIGNUPS_ALLOWED=false` in the environment block

To lock down registration after creating your account, add to the compose environment:

```yaml
environment:
  SIGNUPS_ALLOWED: "false"
```

Then `docker compose up -d` to apply.

## Admin panel

Vaultwarden includes an admin panel at `/admin`. To enable it, set an admin token:

```yaml
environment:
  ADMIN_TOKEN: "your-secure-random-token"
```

Access at `https://vault.yourdomain.com/admin`.

## Backup

The entire vault lives in the `vaultwarden-data` volume. To back up:

```bash
docker run --rm \
  -v vaultwarden-data:/data:ro \
  -v /your/backup/dir:/backup \
  alpine tar czf /backup/vaultwarden-$(date +%F).tar.gz -C /data .
```

Restore by creating a fresh volume and extracting the archive into it before starting the container.
