# 09 — Watchtower

Automatically pulls updated images and restarts containers when a newer version is available. Deploy this **last** — after all other stacks are running and confirmed healthy.

## Services

| Container | Image | Role |
|---|---|---|
| `watchtower` | `containrrr/watchtower` | Automatic container updater |

## Compose file

```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - WATCHTOWER_SCHEDULE=0 0 4 * * *
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_INCLUDE_STOPPED=false
      - TZ=Europe/Prague
```

## Deploy

```bash
cd /home/dfs/stacks/watchtower
docker compose up -d
```

## Schedule

The cron expression `0 0 4 * * *` runs at **04:00 every day** (server local time, set via `TZ`). Watchtower checks for new images, pulls them, and recreates any container whose image has changed.

`WATCHTOWER_CLEANUP=true` removes the old image after a successful update so disk does not accumulate stale layers.

`WATCHTOWER_INCLUDE_STOPPED=false` skips containers that are not currently running — this prevents accidentally pulling updates for services you have intentionally stopped.

## Changing the schedule

The schedule uses a 6-field cron format: `seconds minutes hours day-of-month month day-of-week`.

| Example | Expression |
|---|---|
| 3:30 AM daily | `0 30 3 * * *` |
| 2:00 AM every Sunday | `0 0 2 * * 0` |
| Every 12 hours | `0 0 */12 * * *` |

## Excluding containers from updates

Add a label to any container you want Watchtower to ignore:

```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=false"
```

Or run Watchtower in opt-in mode (`WATCHTOWER_LABEL_ENABLE=true`) and only update containers that explicitly have the label set to `true`.

## Notifications

Watchtower can send update notifications. Add to the environment block:

```yaml
# Shoutrrr URL — supports Slack, Telegram, Discord, email, and more
- WATCHTOWER_NOTIFICATION_URL=telegram://token@telegram?channels=@channelname
```

See the [Shoutrrr docs](https://containrrr.dev/shoutrrr/services/overview/) for all supported providers and URL formats.
