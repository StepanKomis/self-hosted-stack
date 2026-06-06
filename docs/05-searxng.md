# 05 — SearXNG

A self-hosted, privacy-respecting meta search engine that aggregates results from dozens of sources without tracking queries or exposing them to search providers.

## Services

| Container | Image | Role |
|---|---|---|
| `searxng_caddy` | `caddy:2-alpine` | TLS termination + reverse proxy |
| `searxng_redis` | `valkey/valkey:8-alpine` | Rate limiting / caching backend |
| `searxng` | `searxng/searxng` | Search engine |

## Network

All three containers run on an isolated internal network (`searxng_internal`). Caddy uses `network_mode: host` to accept external connections directly on the host's network interfaces — there is no Docker port mapping for it.

## Compose file

```yaml
networks:
  searxng_internal:
    name: searxng_internal
    internal: true

volumes:
  caddy_data:
  caddy_config:
  valkey_data:
  searxng_data:

services:
  caddy:
    image: docker.io/library/caddy:2-alpine
    container_name: searxng_caddy
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    environment:
      - SEARXNG_HOSTNAME=${SEARXNG_HOSTNAME:-http://localhost}
      - SEARXNG_TLS=${LETSENCRYPT_EMAIL:-internal}

  redis:
    image: docker.io/valkey/valkey:8-alpine
    container_name: searxng_redis
    command: valkey-server --save 30 1 --loglevel warning
    restart: unless-stopped
    networks:
      - searxng_internal
    volumes:
      - valkey_data:/data
    mem_limit: 128m

  searxng:
    image: docker.io/searxng/searxng:latest
    container_name: searxng
    restart: unless-stopped
    networks:
      - searxng_internal
    ports:
      - "127.0.0.1:8888:8080"
    volumes:
      - ./searxng:/etc/searxng:rw
      - searxng_data:/var/cache/searxng:rw
    environment:
      - SEARXNG_BASE_URL=https://${SEARXNG_HOSTNAME:-localhost}/
    mem_limit: 512m
```

## Before deploying — required files

### 1. Caddyfile

Create `searxng/Caddyfile` in the stack directory:

```caddy
{$SEARXNG_HOSTNAME} {
    tls {$SEARXNG_TLS}

    @api {
        path /config
        path /healthz
        path /stats/errors
        path /stats/checker
    }

    @search {
        path /search
    }

    @static {
        path /static/*
    }

    header {
        # Security headers
        X-Content-Type-Options "nosniff"
        X-XSS-Protection "1; mode=block"
        X-Frame-Options "SAMEORIGIN"
        Permissions-Policy "accelerometer=(),camera=(),geolocation=(),gyroscope=(),magnetometer=(),microphone=(),payment=(),usb=()"
        Referrer-Policy "no-referrer"
    }

    header @api {
        Access-Control-Allow-Methods "GET, OPTIONS"
        Access-Control-Allow-Origin "*"
    }

    encode zstd gzip

    reverse_proxy localhost:8888
}
```

### 2. SearXNG config directory

```bash
mkdir -p /home/dfs/stacks/searxng/searxng
```

SearXNG will generate `settings.yml` on first boot if it is not present. To pre-configure, create `searxng/settings.yml` based on the [upstream example](https://github.com/searxng/searxng/blob/master/searx/settings.yml).

At minimum, set a secret key:

```yaml
server:
  secret_key: "your-random-secret-key-here"
```

Generate one with:

```bash
openssl rand -hex 32
```

## `.env` additions

Add to your root `.env`:

```env
SEARXNG_HOSTNAME=search.yourdomain.com
LETSENCRYPT_EMAIL=you@example.com
```

## Deploy

```bash
cd /home/dfs/stacks/searxng
docker compose up -d
```

Caddy will request a Let's Encrypt certificate automatically if `SEARXNG_HOSTNAME` is a public domain and port 80/443 are reachable.

For LAN-only use, set `SEARXNG_TLS=internal` (Caddy issues a self-signed cert) and `SEARXNG_HOSTNAME=http://search.local`.

## Verify

```bash
curl -s http://localhost:8888 | grep -o "<title>.*</title>"
```

## Resource limits

SearXNG can be killed by the OOM killer under heavy load. The compose file sets `mem_limit: 512m` which is sufficient for personal use. Raise it if you serve multiple users simultaneously.
