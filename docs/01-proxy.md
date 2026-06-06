# 01 — Nginx Proxy Manager

**Deploy this first.** NPM creates the shared `nginx` Docker network that every other stack attaches to for reverse-proxy access.

## Services

| Container | Image | Role |
|---|---|---|
| `proxy` | `jc21/nginx-proxy-manager` | Reverse proxy, SSL termination |
| `proxy_db` | `mariadb:10.11` | NPM's config database |

## Networks created

| Network | Type | Purpose |
|---|---|---|
| `nginx` | bridge | All proxied services attach here |
| `proxy_external` | bridge | External-facing (port 80/443) |
| `proxy_proxy_internal` | internal | NPM ↔ MariaDB only |

## Host directories required

```bash
mkdir -p /opt/proxy/{data,letsencrypt,mysql}
```

## Compose file

```yaml
services:
  proxy:
    image: jc21/nginx-proxy-manager:latest
    container_name: proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - /opt/proxy/data:/data
      - /opt/proxy/letsencrypt:/etc/letsencrypt
    networks:
      - nginx
      - proxy_external
      - proxy_proxy_internal
    depends_on:
      - proxy_db

  proxy_db:
    image: mariadb:10.11
    container_name: proxy_db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: npm
      MYSQL_DATABASE: npm
      MYSQL_USER: npm
      MYSQL_PASSWORD: npm
    volumes:
      - /opt/proxy/mysql:/var/lib/mysql
    networks:
      - proxy_proxy_internal

networks:
  nginx:
    name: nginx
    driver: bridge
  proxy_external:
    name: proxy_external
    driver: bridge
  proxy_proxy_internal:
    name: proxy_proxy_internal
    internal: true
```

## Deploy

```bash
cd /home/dfs/stacks/proxy
docker compose up -d
```

## First-run setup

1. Open `http://<server-ip>:81`
2. Log in with the default credentials:
   - Email: `admin@example.com`
   - Password: `changeme`
3. You are prompted to change both immediately — do so.

## Adding a proxy host (general pattern)

For every service you want to expose:

1. **Hosts → Proxy Hosts → Add Proxy Host**
2. **Domain Names** — your (sub)domain or `.local` hostname
3. **Forward Hostname/IP** — the container name (e.g. `jellyfin`) or `host` if the service uses `network_mode: host`
4. **Forward Port** — the container's internal port
5. **SSL tab** — request a Let's Encrypt certificate (requires a public domain with DNS pointing to your server), or leave blank for LAN-only `.local` addresses

## Ports summary

| Port | Purpose |
|---|---|
| 80 | HTTP (auto-redirects to HTTPS when SSL is configured) |
| 81 | NPM admin UI |
| 443 | HTTPS |

## Backup

NPM's entire state lives in two directories:

```
/opt/proxy/data/       # proxy rules, SSL certs
/opt/proxy/letsencrypt/
/opt/proxy/mysql/      # MariaDB — hosts, access lists, users
```

Tar all three to restore onto a new host without reconfiguring anything.
