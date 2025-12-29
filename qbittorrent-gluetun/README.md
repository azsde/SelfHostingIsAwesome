# qBittorrent + gluetun (docker-compose)

This repository contains a `docker-compose.yml` to run a qBittorrent instance routed through a VPN provided by a `gluetun` container.

## Prerequisites

- `docker` and `docker compose` installed and working on the host.
- A [supported](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers) VPN provider and credentials (or configuration files) for `gluetun`.

## Quick start

1. Copy the sample env file:

```bash
cp .env.sample .env
```

2. Edit `.env` and fill the required values. Typical variables you must provide:

- `VPN_PROVIDER` — your VPN provider name (or `custom` if using config files).
- `OPENVPN_USER` / `OPENVPN_PASSWORD` — username/password for providers that use them.
- Any provider-specific vars listed in `.env.sample` (region, server, etc.).

3. Configure volumes

- You MUST update the volume mounts in `docker-compose.yml` to point to host folders where you want qBittorrent downloads and configuration, and where any `gluetun` files should persist. This step is mandatory: without correctly set host paths you will either lose downloads/config on container recreation or the services may fail to start with permission/path errors.

- What to change:
    - Locate the `volumes:` section for each service (`qbittorrent`, `gluetun`) in `docker-compose.yml` and replace the example host paths with absolute paths on your machine.
    - Use the format `/<host/path>:<container/path>` for each mount.

- Typical mounts to configure:

```yaml
services:
  qbittorrent:
    volumes:
      - /home/youruser/qbittorrent/config:/config
      - /home/youruser/qbittorrent/downloads:/data

  gluetun:
    volumes:
    - /home/youruser/gluetun:/gluetun
```

- Important notes:
    - Always use absolute host paths (no `~`).

    - Set ownership/permissions so the containers can read/write (UID/GID depends on image).

    - If you fail to configure these volumes properly, downloads won't persist and qBittorrent may not be able to write its settings or accept files.

    - After updating paths, save `docker-compose.yml` and restart the stack with `docker compose up -d`.

4. Start the stack:

```bash
docker compose up -d
```

5. Check logs to confirm `gluetun` connected and qBittorrent started:

```bash
docker compose logs -f gluetun
docker compose logs -f qbittorrent
```

## Accessing qBittorrent Web UI

- Open the qBittorrent Web UI at `http://<host-ip>:<webui-port>` (replace `<webui-port>` with the port defined in `docker-compose.yml` or `.env`).
- If qBittorrent web UI username/password are not defined in environment variables, consult the logs to see which temporary password has been assigned to the `admin` user.

## Port forwarding

- This setup enables port forwarding when the VPN provider supports it. `gluetun` will request a forwarded (dedicated) port from the VPN provider and make that port available to the container.
- How it works (high level):
    1. `gluetun` asks the VPN provider for a forwarded port after establishing the tunnel.
    2. The provider responds with a port number that accepts incoming connections mapped to your VPN session.
    3. `gluetun` exposes that forwarded port to other services in the stack (commonly via an environment variable, a file, or through the Docker compose configuration depending on how the compose is written).
    4. The qBittorrent service is configured to use that forwarded port as its incoming (listening) port so it can accept direct peer connections.

- Why this matters: a dedicated forwarded port improves incoming connectivity (more peers, better swarm performance) while keeping your public IP hidden behind the VPN.

- What you should check after starting the stack:
    - Inspect the `gluetun` logs for a message like "Forwarded port" or similar to see the assigned port.
    - Confirm qBittorrent's "Listening Port" (in the Web UI or config file) matches the forwarded port. The compose in this repository is arranged to set qBittorrent to use the forwarded port automatically — if you don't see that, I can update the compose to perform the automatic mapping.

- Notes:
    - Not all VPN providers support port forwarding. If your provider does not, no forwarded port will be assigned.
    - When `network_mode: service:gluetun` is used, qBittorrent shares gluetun's network namespace and benefits from the forwarded port without additional host port mappings. If not using that mode, ensure any required port is exposed correctly in the compose file.

## Troubleshooting

- If torrents do not download or web UI is inaccessible:
    - Verify `gluetun` shows a connected VPN in its logs.
    - Confirm qBittorrent is running and listening on the expected port.
    - If using `network_mode: service:gluetun`, the web UI port must be exposed from the gluetun container or proxied — check compose file.

## Security and privacy

- Ensure DNS leaks are prevented by verifying `gluetun` DNS settings.
- Do not expose qBittorrent's Web UI to untrusted networks without authentication and HTTPS.
- You can check that everything is working using services such as: https://ipleak.net/ or https://torguard.net/check-my-torrent-ip-address/
