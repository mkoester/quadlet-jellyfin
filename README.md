# quadlet-jellyfin

Quadlet setup for [Jellyfin](https://jellyfin.org/) — an open-source media server (`docker.io/jellyfin/jellyfin`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `jellyfin.container` | Quadlet unit file |
| `jellyfin.env` | Default environment variables |
| `jellyfin.override.env.template` | Template for local overrides (public URL, timezone) |
| `jellyfin-backup.service` | Systemd service: SQLite snapshots + rsync of remaining config |
| `jellyfin-backup.timer` | Systemd timer: triggers the backup daily |

## Setup

### Add media directories

Edit `jellyfin.container` and add a `Volume=` line for each media location before running the service:

```ini
Volume=/mnt/media/movies:/media/movies:ro
Volume=/mnt/media/tv:/media/tv:ro
```

The `jellyfin` service user must be able to read those host paths (see [Media access](#media-access) below). Do **not** add `:Z` to shared media mounts.

### Service setup

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/jellyfin -s /usr/sbin/nologin jellyfin

REPO_URL=https://github.com/mkoester/quadlet-jellyfin.git
REPO=~jellyfin/quadlet-jellyfin
```

```sh
# 2. Enable linger
sudo loginctl enable-linger jellyfin

# 3. Clone this repo into the service user's home
sudo -u jellyfin git clone $REPO_URL $REPO

# 4. Create quadlet, config, and cache directories
sudo -u jellyfin mkdir -p ~jellyfin/.config/containers/systemd
sudo -u jellyfin mkdir -p ~jellyfin/{config,cache}

# 5. Create .override.env from template and fill in required values
sudo -u jellyfin cp $REPO/jellyfin.override.env.template $REPO/jellyfin.override.env
sudo -u jellyfin nano $REPO/jellyfin.override.env

# 6. Symlink all quadlet files from the repo
sudo -u jellyfin ln -s $REPO/jellyfin.container ~jellyfin/.config/containers/systemd/jellyfin.container
sudo -u jellyfin ln -s $REPO/jellyfin.env ~jellyfin/.config/containers/systemd/jellyfin.env
sudo -u jellyfin ln -s $REPO/jellyfin.override.env ~jellyfin/.config/containers/systemd/jellyfin.override.env

# 7. Reload and start
sudo -u jellyfin XDG_RUNTIME_DIR=/run/user/$(id -u jellyfin) systemctl --user daemon-reload
sudo -u jellyfin XDG_RUNTIME_DIR=/run/user/$(id -u jellyfin) systemctl --user start jellyfin

# 8. Verify
sudo -u jellyfin XDG_RUNTIME_DIR=/run/user/$(id -u jellyfin) systemctl --user status jellyfin
```

## Configuration

### Environment variables

`jellyfin.env` contains the defaults:

| Variable | Default | Description |
|---|---|---|
| `TZ` | `Europe/Berlin` | Container timezone |

`jellyfin.override.env` (created from template) must set:

| Variable | Description |
|---|---|
| `JELLYFIN_PublishedServerUrl` | Public HTTPS URL (e.g. `https://jellyfin.example.com`) — needed for autodiscovery behind a reverse proxy |

To apply changes after editing the override file:

```sh
sudo -u jellyfin nano ~jellyfin/quadlet-jellyfin/jellyfin.override.env
sudo -u jellyfin XDG_RUNTIME_DIR=/run/user/$(id -u jellyfin) systemctl --user restart jellyfin
```

### Media access

The `jellyfin` service user must be able to read media files on the host. Add it to whatever group owns the media directories:

```sh
sudo usermod -aG <media-group> jellyfin
```

Or, if media directories are world-readable, no extra step is needed.

### Hardware transcoding (optional)

For Intel/AMD GPU transcoding, uncomment the `AddDevice=` line in `jellyfin.container`:

```ini
AddDevice=/dev/dri/renderD128
```

Then add the `jellyfin` user to the `render` (and optionally `video`) group on the host:

```sh
sudo usermod -aG render jellyfin
sudo usermod -aG video jellyfin
```

Enable hardware acceleration in the Jellyfin admin UI under *Dashboard → Playback → Transcoding*.

## Reverse proxy (Caddy)

Add a site block to your Caddyfile:

```
jellyfin.example.com {
    reverse_proxy localhost:8096
}
```

After editing the Caddyfile, reload Caddy:

```sh
sudo systemctl reload caddy
```

## Backup

Jellyfin stores configuration and metadata under `~jellyfin/config/` on the host. The two main SQLite databases are snapshotted with `sqlite3 .backup` for consistency; remaining config files are synced with `rsync`. See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) for the one-time server-wide setup.

```sh
# 1. Create backup staging directories (owned by jellyfin, readable by backup-readers group)
sudo mkdir -p /var/backups/jellyfin/config
sudo chown -R jellyfin:backup-readers /var/backups/jellyfin
sudo chmod -R 750 /var/backups/jellyfin

# 2. Symlink the backup service and timer from the repo
sudo -u jellyfin mkdir -p ~jellyfin/.config/systemd/user
sudo -u jellyfin ln -s $REPO/jellyfin-backup.service ~jellyfin/.config/systemd/user/jellyfin-backup.service
sudo -u jellyfin ln -s $REPO/jellyfin-backup.timer ~jellyfin/.config/systemd/user/jellyfin-backup.timer

# 3. Enable and start the timer
sudo -u jellyfin XDG_RUNTIME_DIR=/run/user/$(id -u jellyfin) systemctl --user daemon-reload
sudo -u jellyfin XDG_RUNTIME_DIR=/run/user/$(id -u jellyfin) systemctl --user enable --now jellyfin-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@jellyfin-host:/var/backups/jellyfin/ /path/to/local/backup/jellyfin/
```

## Notes

- Port `8096` is bound to `127.0.0.1` only — Caddy handles TLS termination.
- All persistent state is at `~jellyfin/config/` and `~jellyfin/cache/` on the host.
- Media files are bind-mounted from the host; they are not stored under the service user's home.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u jellyfin XDG_RUNTIME_DIR=/run/user/$(id -u jellyfin) systemctl --user enable --now podman-auto-update.timer
  ```
- To prune old images automatically, enable the system-wide prune timer (see [image pruning setup](https://github.com/mkoester/quadlet-my-guidelines#image-pruning)). Replace `30` with the desired retention period in days:
  ```sh
  sudo -u jellyfin XDG_RUNTIME_DIR=/run/user/$(id -u jellyfin) systemctl --user enable --now podman-image-prune@30.timer
  ```
