# Jellyfin Homelab

Self-hosted [Jellyfin](https://jellyfin.org/) media server running on Docker Compose,
with GPU-accelerated transcoding (VAAPI / Intel Quick Sync / AMD) out of the box.

## Features

- 🐳 Single-service Docker Compose setup
- 🎞️ Hardware-accelerated transcoding via `/dev/dri` — low CPU usage during playback
- 🔒 Runs as a non-root user (`1000:1000`)
- 🧩 Media paths configured through a `.env` file — no need to edit the compose file
- 📡 LAN auto-discovery (UDP `7359`) for Jellyfin clients
- 🔐 Remote access through [Tailscale](https://tailscale.com/) — no ports forwarded on the router

## Requirements

- Linux host with Docker and Docker Compose v2
- Intel or AMD GPU exposing `/dev/dri` (optional, for hardware transcoding)
- [Tailscale](https://tailscale.com/) installed on the host and on client devices
  (optional, for remote access)

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

   On Intel systems with more than one GPU, check which render node belongs to the iGPU
   before picking the device:

   ```bash
   ls -l /dev/dri/by-path/
   ```

## Remote Access (Tailscale)

Devices on your tailnet can reach Jellyfin directly, so nothing needs to be exposed to
the public internet.

```bash
# Start the daemon and enable it on boot
sudo systemctl enable --now tailscaled

# Join your tailnet (prints a login URL)
sudo tailscale up
```

Then:

1. Find the tailnet address of the server:

   ```bash
   tailscale ip -4
   ```

2. In Jellyfin, go to **Dashboard → Networking → LAN networks** and add the Tailscale
   range `100.64.0.0/10`. Without this, Jellyfin treats tailnet clients as remote
   connections and applies remote bitrate limits, which forces needless transcoding.

3. Install Tailscale on the client device (phone, laptop, Android TV) and open:

   ```
   http://<tailscale-ip>:8096
   ```

Optional — serve Jellyfin under an HTTPS tailnet hostname instead of a raw IP. MagicDNS
and HTTPS certificates must be enabled in the Tailscale admin console first:

```bash
sudo tailscale serve --bg --https=443 http://127.0.0.1:8096
tailscale serve status
```

Tailscale runs on the host, not in Docker. Its state (`/var/lib/tailscale`) and keys are
machine-local and deliberately kept out of this repository.

## Ports

| Port         | Protocol | Purpose                       |
| ------------ | -------- | ----------------------------- |
| `8096`       | TCP      | Web UI and API                |
| `7359`       | UDP      | Client auto-discovery on LAN  |

Neither port needs to be forwarded on the router: LAN clients reach the host directly and
remote clients connect over Tailscale.

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
- **`FFmpeg exited with code 251`** — the VA-API device points at a GPU with no VAAPI
  driver in the container (commonly a discrete NVIDIA card). Check
  `ls -l /dev/dri/by-path/` and select the Intel render node in
  **Dashboard → Playback → Transcoding → VA-API Device**.
- **Empty library** — double-check `MOVIES_PATH` / `SERIES_PATH` in `.env` point to real
  host directories, then rescan libraries in Jellyfin.
- **Permission errors** — the container runs as `1000:1000`; make sure `config/` and
  `cache/` are owned by that user.
- **Cannot reach the server over Tailscale** — confirm `tailscale status` shows the host
  as `online` and that `tailscaled` is running (`systemctl status tailscaled`).
- **Transcoding starts on tailnet clients for no reason** — `100.64.0.0/10` is missing
  from **Dashboard → Networking → LAN networks**.
