# media-server

Docker Compose stacks for a Fedora Workstation media server, managed by [Komodo](https://komo.do).

Each directory is an independent Compose project. They all attach to one shared
external Docker network so containers can reach each other by service name.

## Stacks

| Directory | Service | Host port |
| --- | --- | --- |
| `plex` | Plex Media Server (NVIDIA transcoding) | 32400 |
| `radarr` | Movie management | 7878 |
| `sonarr` | TV management | 8989 |
| `bazarr` | Subtitles | 6767 |
| `prowlarr` | Indexer manager | 9696 |
| `seerr` | Request portal (Overseerr) | 5055 |
| `tautulli` | Plex statistics | 8181 |
| `qbittorrent` | Torrent client | 8080 |
| `deluge` | Torrent client | 8112 |
| `cloudflare-tunnel` | Public ingress for Seerr | — |
| `komodo` | Stack management UI | 9120 |
| `monitoring` | Prometheus, Grafana, Loki, Alertmanager | 9090, 3000, 9093 |

## First-time setup

1. Create the shared network:

   ```bash
   docker network create media
   ```

2. Create the media layout. Everything must live under a single root on one
   filesystem so Radarr/Sonarr can hardlink instead of copying:

   ```bash
   export DATA_ROOT=/mnt/media
   mkdir -p "$DATA_ROOT"/{movies,tvseries}
   mkdir -p /home/alexandrung/docker/config
   ```

3. Copy the env template in each stack directory and fill it in:

   ```bash
   for d in */; do [ -f "$d/.env.example" ] && cp -n "$d/.env.example" "$d/.env"; done
   ```

   `.env` files are gitignored. `.env.example` at the repo root lists every
   variable used anywhere.

4. Bring up the stacks, or point Komodo at this repo and deploy from the UI:

   ```bash
   docker compose -f komodo/docker-compose.yml up -d
   ```

## Combined stacks for Komodo

`stacks/` holds the same services collapsed into one file each, for pasting
straight into a Komodo Stack:

| File | Contains |
| --- | --- |
| `stacks/media.compose.yaml` | Plex, Radarr, Sonarr, Bazarr, Prowlarr, Seerr, Tautulli, qBittorrent, Deluge, Cloudflare Tunnel |
| `stacks/monitoring.compose.yaml` | Prometheus, Grafana, Node Exporter, Loki, Promtail, Alertmanager, exportarr |

The matching `*.env.example` goes into the Stack's **Environment** box. The
monitoring file inlines every config through Compose `configs`, so it needs no
files on disk.

The per-directory compose files remain the source of truth if you prefer to run
stacks individually with `docker compose`. Use one approach or the other, not
both at once — container names collide.

### Auto-updating images

Komodo does **not** update images on its own by default. Per Stack, enable:

- **Poll For Updates** — checks the registry and flags newer images in the UI.
- **Auto Update** — pulls and redeploys automatically when a newer image lands.

This only works with moving tags. Everything here uses `:latest` except Loki
and Promtail, which are pinned to `2.9.2` because their config format changes
between minor versions.

## Path convention

One flat tree. Downloads land directly in the library folders that Plex reads,
so nothing is ever moved or copied between locations:

```
/data
├── movies          <- deluge downloads here, radarr manages, plex reads
└── tvseries        <- qbittorrent downloads here, sonarr manages, plex reads
```

The same path is used inside and outside every container:

| Container | Mount | Sees |
| --- | --- | --- |
| deluge | `${DATA_ROOT}/movies` | `/data/movies` |
| qbittorrent | `${DATA_ROOT}/tvseries` | `/data/tvseries` |
| radarr | `${DATA_ROOT}` | `/data` |
| sonarr | `${DATA_ROOT}` | `/data` |
| bazarr | `${DATA_ROOT}` | `/data` |
| plex | `${DATA_ROOT}` (read-only) | `/data` |

Settings to match:

- Deluge download directory: `/data/movies`
- qBittorrent default save path: `/data/tvseries`
- Radarr root folder: `/data/movies`
- Sonarr root folder: `/data/tvseries`
- Plex libraries: `/data/movies` and `/data/tvseries`

Radarr and Sonarr see the download client's files at the same path they manage,
so imports are instant renames rather than copies.

## Monitoring

- **Grafana** — <http://localhost:3000>, credentials come from `monitoring/.env`.
  Prometheus, Loki, and Alertmanager data sources are provisioned automatically.
- **Prometheus** — <http://localhost:9090>, scrapes Node Exporter and exportarr
  sidecars for Radarr, Sonarr, and Prowlarr.
- **Loki + Promtail** — all container logs, queryable in Grafana.
- **Alertmanager** — <http://localhost:9093>. Rules live in
  `monitoring/alert-rules.yml`; add a receiver in `monitoring/alertmanager.yml`
  or alerts go nowhere.

Exporters are not published to the host; they are reachable only on the
`media` network.

## Security notes

- Never commit a `.env`. The `.gitignore` covers `*.env` but excludes
  `*.env.example`.
- Only Seerr is exposed publicly, via the Cloudflare tunnel.
- Enable authentication in every web UI before exposing anything.
- Komodo Periphery has access to the Docker socket, which is equivalent to root
  on the host. Keep its port off the public internet.
