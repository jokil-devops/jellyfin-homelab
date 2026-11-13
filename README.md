# Jellyfin Homelab

Self-hosted [Jellyfin](https://jellyfin.org/) media server running on Docker Compose,
with GPU-accelerated transcoding (VAAPI / Intel Quick Sync / AMD) out of the box.

## Features

- 🐳 Single-service Docker Compose setup
- 🎞️ Hardware-accelerated transcoding via `/dev/dri` — low CPU usage during playback
- 🔒 Runs as a non-root user (`1000:1000`)
- 🧩 Media paths configured through a `.env` file — no need to edit the compose file
- 📡 LAN auto-discovery (UDP `7359`) for Jellyfin clients

## Requirements

- Linux host with Docker and Docker Compose v2
- Intel or AMD GPU exposing `/dev/dri` (optional, for hardware transcoding)

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/jokil-devops/jellyfin-homelab.git
cd jellyfin-homelab

# 2. Create your .env file
cp .env.example .env
# then edit .env and set your timezone and media paths

# 3. Start the server
docker compose up -d
```

Open the web interface at `http://<host-ip>:8096` and complete the initial setup wizard.

## Configuration

All settings live in `.env`:

| Variable      | Description                          | Example              |
| ------------- | ------------------------------------ | -------------------- |
| `TZ`          | Timezone of the container            | `Europe/London`      |
| `MOVIES_PATH` | Host path to your movies directory   | `/home/user/films`   |
| `SERIES_PATH` | Host path to your series directory   | `/home/user/series`  |

Both media directories are mounted into the container **read-only** (`/media/movies`, `/media/series`).

## Hardware Transcoding

The container passes `/dev/dri` through to Jellyfin. Two things to check:

1. **Render group GID** — the compose file uses `group_add: "987"`. Make sure it matches
   the `render` group on your host:

   ```bash
   getent group render
   ```

   Update `group_add` in `docker-compose.yaml` if the GID differs.

2. **Enable it in Jellyfin** — go to **Dashboard → Playback → Transcoding** and set
   *Hardware acceleration* to **Video Acceleration API (VAAPI)**.

## Ports

| Port         | Protocol | Purpose                       |
| ------------ | -------- | ----------------------------- |
| `8096`       | TCP      | Web UI and API                |
| `7359`       | UDP      | Client auto-discovery on LAN  |

## Repository Structure

```
.
├── docker-compose.yaml   # Jellyfin service definition
├── .env.example          # Template for environment configuration
└── README.md
```

Runtime data (`config/`, `cache/`) and media directories are created at run time and
excluded from version control via `.gitignore`.

## Troubleshooting

- **No hardware transcoding** — verify `/dev/dri` exists on the host and that your user's
  `group_add` GID matches the `render` group.
- **Empty library** — double-check `MOVIES_PATH` / `SERIES_PATH` in `.env` point to real
  host directories, then rescan libraries in Jellyfin.
- **Permission errors** — the container runs as `1000:1000`; make sure `config/` and
  `cache/` are owned by that user.
