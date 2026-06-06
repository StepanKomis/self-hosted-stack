# 08 — Immich Machine Learning (Sidecar)

A standalone machine-learning sidecar for an Immich photo server running on a separate host. This stack exposes only the inference API on port 3003 — the Immich main app connects to it remotely.

If your full Immich instance runs on the same host, use the [official Immich compose file](https://immich.app/docs/install/docker-compose) instead of this sidecar-only setup.

## Services

| Container | Image | Role | Port |
|---|---|---|---|
| `immich-ml-immich-machine-learning-1` | `immich-app/immich-machine-learning:release` | ML inference (face recognition, CLIP, etc.) | 3003 |

## Volumes

| Volume | Content |
|---|---|
| `model_cache` | Downloaded CLIP and facial-recognition models |

Models are downloaded on first use and cached — first requests after a fresh deploy will be slow.

## Compose file

```yaml
volumes:
  model_cache:

services:
  immich-machine-learning:
    image: ghcr.io/immich-app/immich-machine-learning:release
    container_name: immich-ml-immich-machine-learning-1
    restart: unless-stopped
    ports:
      - "3003:3003"
    volumes:
      - model_cache:/cache
```

## Deploy

```bash
cd /home/dfs/stacks/immich-ml
docker compose up -d
```

## Connect to a remote Immich instance

On the host running the **main Immich app**, edit its `docker-compose.yml` or environment to point the ML URL at this server:

```env
IMMICH_MACHINE_LEARNING_URL=http://<this-server-ip>:3003
```

Verify the sidecar is reachable from the remote host:

```bash
curl http://<this-server-ip>:3003/ping
# expected: {"res":"pong"}
```

## GPU acceleration (optional)

The default image runs ML inference on CPU. For GPU acceleration, see the [Immich hardware transcoding docs](https://immich.app/docs/features/ml-hardware-acceleration):

- **CUDA (NVIDIA):** use image tag `:cuda` and add `deploy.resources.reservations.devices`
- **OpenVINO (Intel):** use image tag `:openvino` and pass `/dev/dri`
- **ROCM (AMD):** use image tag `:rocm` and pass `/dev/dri` + `/dev/kfd`

## Notes

- No `nginx` network is needed — this sidecar is not proxied, it is called server-to-server on port 3003
- The container name is intentionally long (`immich-ml-immich-machine-learning-1`) to match what Portainer/Immich expects when referencing the sidecar by name
