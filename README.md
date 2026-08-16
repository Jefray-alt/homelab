# Homelab Docker Compose

A self-hosted media stack managed with a single `docker-compose.yaml`.

## Services

| Service | Role | Image | Port |
|---------|------|-------|------|
| [Jellyfin](https://jellyfin.org) | Media server (TV & movies) | `lscr.io/linuxserver/jellyfin` | 8096 |
| [Sonarr](https://sonarr.tv) | TV show download manager | `lscr.io/linuxserver/sonarr` | 8989 |
| [Radarr](https://radarr.video) | Movie download manager | `lscr.io/linuxserver/radarr` | 7878 |
| [Prowlarr](https://prowlarr.com) | Indexer manager | `lscr.io/linuxserver/prowlarr` | 9696 |
| [qBittorrent](https://www.qbittorrent.org) | Download client (Torrents) | `lscr.io/linuxserver/qbittorrent` | 8080 (Web UI), 6881 (TCP/UDP) |
| [Seerr](https://github.com/seerr-team/seerr) | Media request management | `ghcr.io/seerr-team/seerr` | 5055 |
| [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr) | Cloudflare/anti-bot bypass proxy | `ghcr.io/flaresolverr/flaresolverr` | 8191 (configurable) |

## Architecture

```
Seerr ──request──▶ Prowlarr ──indexers──▶ Sonarr/Radarr
                                               │
                                               ▼
                                        qBittorrent ──download──▶ /mnt/media
                                               │                        │
                                               └──────────◀──────────────┘ (hardlinks)
                                               ▼
                                           Jellyfin ──stream──▶ your devices
```

- **Seerr** lets users request shows/movies.
- **Prowlarr** aggregates torrent indexers for Sonarr and Radarr.
- **Sonarr/Radarr** manage the library and send downloads to qBittorrent.
- **qBittorrent** downloads to `/mnt/media`, and Sonarr/Radarr hardlink completed torrents into the organized media structure so Jellyfin can play them.
- **FlareSolverr** is wired into Prowlarr's indexers to bypass Cloudflare/anti-bot protection.
- **Jellyfin** serves the media to clients.

## Getting Started

1. Create the config/data directories (or edit the volume paths in the compose file):

```bash
sudo mkdir -p /docker/appdata/{jellyfin,qbittorrent,prowlarr,sonarr,radarr,seerr,flaresolver}
```

2. (Optional) Create a `.env` file to override defaults:

```env
LOG_LEVEL=info
PORT=8191
```

3. Start the stack:

```bash
docker compose up -d
```

4. Follow the first-run setup for each service via its web UI.

## First-Run Setup

1. **Jellyfin** (`http://<host>:8096`) — complete the media library wizard, pointing at `/data/media` (mapped from `/mnt/media`).
2. **qBittorrent** (`http://<host>:8080`, default user `admin`, password is in the container logs) — optional: set download path to `/data`.
3. **Prowlarr** (`http://<host>:9696`) — add indexers, configure FlareSolverr at `http://<host>:8191`, then add Prowlarr as indexer in Sonarr/Radarr.
4. **Sonarr** (`http://<host>:8989`) — add root folder `/data`, connect to qBittorrent and Prowlarr.
5. **Radarr** (`http://<host>:7878`) — same as Sonarr, for movies.
6. **Seerr** (`http://<host>:5055`) — sign in, link Sonarr and Radarr.

## Ports

| Port | Service | Notes |
|------|---------|-------|
| 5055 | Seerr | |
| 6881 | qBittorrent | Torrent traffic, TCP + UDP |
| 7878 | Radarr | |
| 8080 | qBittorrent | Web UI |
| 8096 | Jellyfin | |
| 8191 | FlareSolverr | Override with `PORT` env var |
| 8989 | Sonarr | |
| 9696 | Prowlarr | |

## Data & Storage

- **Config**: All app config is persisted under `/docker/appdata/<service>`.
- **Media**: `/mnt/media` is mounted as `/data` in the downloader and *-arr containers so torrents can be **hardlinked** into the media library (saves disk space, no double-copying). Jellyfin mounts it **read-only** (`/data:ro`).
- Ensure all containers use the same `PUID`/`PGID` (1000 here) so file permissions match across services.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `LOG_LEVEL` | `info` | FlareSolverr log level |
| `LOG_FILE` | `none` | FlareSolverr log file path |
| `LOG_HTML` | `false` | FlareSolverr HTML logging |
| `CAPTCHA_SOLVER` | `none` | FlareSolverr captcha solver |
| `PORT` | `8191` | FlareSolverr host port |

## Troubleshooting

- **Torrents stuck / no peers**: FlareSolverr is only for bypassing web indexer protection, not for torrent traffic — make sure port 6881 (TCP/UDP) is forwarded on your router.
- **Permission errors**: Keep `PUID`/`PGID` identical across containers and verify the media mount owner.
- **Seerr healthcheck failing**: Seerr may take up to 20s to start; the healthcheck uses a `start_period` of 20s.

## Useful Commands

```bash
docker compose ps        # status of all services
docker compose logs -f   # follow logs for all services
docker compose logs sonarr
docker compose up -d     # start / update after config change
docker compose down      # stop all services
docker compose pull && docker compose up -d   # update images
```

## Notes

- Host timezone is set to `Asia/Manila` (`TZ`).
- FlareSolverr and Prowlarr use explicit public DNS (1.1.1.1, 8.8.8.8) to resolve indexers.
- Some images reference the LinuxServer.io registry (`lscr.io`) and ghcr.io — a Docker Hub mirror is not required.