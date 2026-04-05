# goingmerry-ansible

Ansible playbook to fully automate the setup of a Raspberry Pi 5 home server running Ubuntu Server 24.04 LTS.

## What it sets up

- **System** — packages, hostname, timezone (Asia/Kolkata), apt upgrade, SSH hardening, cgroup memory for Docker
- **Users** — `luffy` (sudo, zsh, oh-my-zsh), `media` and `monitoring` service users, `media-data` and `docker` groups
- **Dotfiles** — oh-my-zsh with robbyrussell theme, .vimrc for luffy
- **Docker** — Docker Engine, `media-net` and `monitoring-net` bridge networks
- **Storage** — exFAT external drive mounted at `/mnt/voidcentury`, correct ownership and permissions, media folder structure
- **NordVPN** — installed on host, token login, LAN discovery, autoconnect
- **Monitoring stack** — InfluxDB 2.7, Grafana, Telegraf (OpenWrt collectd), Telegraf (Pi host metrics + Docker)
- **Media stack** — Jellyfin, qBittorrent, Sonarr, Radarr, Prowlarr
- **Caddy** — reverse proxy in Docker, bridges both networks, serves all services on `*.goingmerry`

## Hardware

- Pi 5 (goingmerry) — Ubuntu Server 24.04 LTS on NVMe, static IP `192.168.1.157`
- Pi 4 (OpenWrt router at `192.168.1.1`) — not touched by this playbook
- 1TB exFAT external drive — mounted at `/mnt/voidcentury`

## Running on the Pi

Flash your NVMe/SD card with Raspberry Pi Imager and set:
- **Username:** `luffy`
- **Hostname:** `goingmerry`
- **SSH:** enabled

Then SSH in and run:

```bash
sudo apt install -y ansible
git clone <repo-url> ~/goingmerry-ansible
cd ~/goingmerry-ansible
ansible-galaxy install -r requirements.yml
ansible-playbook goingmerry.yml -i inventory.yml --ask-vault-pass
```

> `ansible-galaxy install` must run before the playbook — the prettify output plugin is loaded by ansible at startup before any tasks run.

Ansible collections are installed automatically at the start of the playbook.

> After the first run, reboot the Pi for cgroup memory changes to take effect.

## Restoring from backups (optional)

If you have existing config backups, place them on the Pi before running the playbook:

```
~/backups/sonarr-config/
~/backups/radarr-config/
~/backups/prowlarr-config/
~/backups/grafana-data/
~/backups/influxdb-backup/
```

The playbook detects and restores them automatically if present.

## Services

All services accessible on the local network via Caddy on port 80:

| Service     | URL                        |
|-------------|----------------------------|
| Jellyfin    | http://tv.goingmerry        |
| qBittorrent | http://cargo.goingmerry     |
| Sonarr      | http://sonarr.goingmerry    |
| Radarr      | http://radarr.goingmerry    |
| Prowlarr    | http://prowlarr.goingmerry  |
| Grafana     | http://grafana.goingmerry   |
| InfluxDB    | http://influxdb.goingmerry  |

> Add `192.168.1.157` entries to your router's DNS or client `/etc/hosts` for `*.goingmerry` to resolve.

## Project structure

```
goingmerry-ansible/
├── inventory.yml               # localhost — runs on the Pi itself
├── goingmerry.yml              # main playbook
├── requirements.yml            # ansible-galaxy dependencies (auto-installed)
├── vars/
│   ├── main.yml                # non-secret variables
│   └── vault.yml               # encrypted secrets
├── roles/
│   ├── system/                 # packages, apt upgrade, hostname, timezone, sshd, cgroup
│   ├── users/                  # luffy, media, monitoring users + groups
│   ├── dotfiles/               # oh-my-zsh, .zshrc, .vimrc
│   ├── docker/                 # docker engine + networks
│   ├── storage/                # fstab, directories, permissions
│   ├── nordvpn/                # install, token auth, LAN discovery, autoconnect
│   ├── monitoring/             # InfluxDB, Grafana, Telegraf x2
│   ├── media/                  # Jellyfin, qBittorrent, Sonarr, Radarr, Prowlarr
│   └── caddy/                  # reverse proxy
└── files/
    └── monitoring/             # telegraf config templates
```

## Notes

- Idempotent — safe to run multiple times
- InfluxDB init (org, admin, token, `network` bucket) is handled by Docker env vars on first start. The `goingmerry` bucket is created via the `influx` CLI post-startup.
- Caddy runs without HTTPS (`auto_https off`) — local network only
- Telegraf listens on UDP 25826 for collectd data from OpenWrt — configure your router to send collectd data to `192.168.1.157`
