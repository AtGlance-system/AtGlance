# AtGlance Management Console: deploy and upgrade

This repository holds only what you need to **install** the AtGlance Management Console (Community
Edition) on a server, or to **upgrade** an existing one: the Docker Compose files, the installer and
updater scripts, and the guides. The console ships as container images; no source code is needed.

**Images:** `ghcr.io/atglance-system/atglance/ce-atglance-app`, `ce-atglance-gateway` and
`ce-atglance-mcp` on the GitHub Container Registry, tag `latest` (console **2.4.0**). The same images are
on Docker Hub as `atglance/ce-atglance-*`.

Every free feature is included. The premium features of this release, **NFS Storage** and **Azure
Storage**, use the same images and switch on with a licence that includes them.

## Documentation (wiki)

Step-by-step guides are in the **[wiki](https://github.com/AtGlance-system/atGlance/wiki)**:

| Get started | Run and maintain | Help |
|---|---|---|
| [Requirements](https://github.com/AtGlance-system/atGlance/wiki/Requirements) | [Upgrade](https://github.com/AtGlance-system/atGlance/wiki/Upgrade) | [Troubleshooting](https://github.com/AtGlance-system/atGlance/wiki/Troubleshooting) |
| [Install on Linux](https://github.com/AtGlance-system/atGlance/wiki/Install-on-Linux) | [Rollback](https://github.com/AtGlance-system/atGlance/wiki/Rollback) | [FAQ](https://github.com/AtGlance-system/atGlance/wiki/FAQ) |
| [Install on Windows](https://github.com/AtGlance-system/atGlance/wiki/Install-on-Windows) | [Backup and Restore](https://github.com/AtGlance-system/atGlance/wiki/Backup-and-Restore) | [Images and Registries](https://github.com/AtGlance-system/atGlance/wiki/Images-and-Registries) |
| [Manual Install](https://github.com/AtGlance-system/atGlance/wiki/Manual-Install) | [Configuration](https://github.com/AtGlance-system/atGlance/wiki/Configuration) | [Optional Features](https://github.com/AtGlance-system/atGlance/wiki/Optional-Features) |
| [After the Install](https://github.com/AtGlance-system/atGlance/wiki/After-the-Install) | [Manage and Uninstall](https://github.com/AtGlance-system/atGlance/wiki/Manage-and-Uninstall) | |
| [Connect Servers](https://github.com/AtGlance-system/atGlance/wiki/Connect-Servers) | | |

## Install

**Linux** (Docker host; the installer installs Docker if it is missing), as root:

```bash
curl -fsSL https://raw.githubusercontent.com/AtGlance-system/atGlance/main/scripts/atglance-installer.sh | sudo bash
```

**Windows** (Docker Desktop in Linux containers mode), PowerShell as Administrator:

```powershell
irm https://raw.githubusercontent.com/AtGlance-system/atGlance/main/scripts/atglance-installer.ps1 | iex
```

The installer checks the host (CPU, memory, Docker, Docker Compose), then runs the updater:
- **new server:** it creates `/opt/atglance` (Windows: `C:\ProgramData\AtGlance`) with a `.env` holding
  long random passwords, pulls the images and starts the console;
- **existing console:** it upgrades it (see below).

Then open `http://<server>:8000`, run the setup wizard (first administrator and organization) and add your
licence under **Settings › Licence**. Get a licence key from your AtGlance dashboard.
Next steps: [After the Install](https://github.com/AtGlance-system/atGlance/wiki/After-the-Install) and [Connect Servers](https://github.com/AtGlance-system/atGlance/wiki/Connect-Servers).

## Upgrade

On a running console:

```bash
curl -fsSL https://raw.githubusercontent.com/AtGlance-system/atGlance/main/scripts/atglance_update.sh | sudo bash
```

```powershell
irm https://raw.githubusercontent.com/AtGlance-system/atGlance/main/scripts/atglance_update.ps1 -OutFile atglance_update.ps1
powershell -ExecutionPolicy Bypass -File .\atglance_update.ps1
```

The updater backs up the database, refreshes the compose files (keeping the overlays the console was
started with), pulls the newest images, recreates the containers and checks the database and their health.
Data, passwords and settings are kept. If a check fails it exits with code 2; the backup folder has a
`ROLLBACK.txt` with the exact commands to go back. Details: [Upgrade](https://github.com/AtGlance-system/atGlance/wiki/Upgrade) and [Rollback](https://github.com/AtGlance-system/atGlance/wiki/Rollback).

**Consoles installed from Docker Hub** keep using Docker Hub (the updater reads the registry of the running
containers). To move one to the GitHub Container Registry (same images), set this in its `.env`
(`/opt/atglance/.env`), then run the updater:

```bash
ATGLANCE_REGISTRY=ghcr.io/atglance-system/atglance
```

More in [Images and Registries](https://github.com/AtGlance-system/atGlance/wiki/Images-and-Registries).

## Manual install with Docker Compose

```bash
sudo git clone https://github.com/AtGlance-system/atGlance.git /opt/atglance
cd /opt/atglance
sudo cp .env.example .env
# set DB_PASSWORD, DB_ROOT_PASSWORD and REDIS_PASSWORD to long random values, for example:
sudo sed -i "s/^DB_PASSWORD=.*/DB_PASSWORD=$(openssl rand -hex 24)/; s/^DB_ROOT_PASSWORD=.*/DB_ROOT_PASSWORD=$(openssl rand -hex 24)/; s/^REDIS_PASSWORD=.*/REDIS_PASSWORD=$(openssl rand -hex 24)/" .env
sudo docker compose up -d
```

Optional overlays, added with `-f`:

```bash
sudo docker compose -f docker-compose.yml -f docker-compose.domain.yml -f docker-compose.nfs.yml up -d
```

To upgrade a manual install later, run the updater above with `--dir /opt/atglance`, or
`git pull && sudo docker compose pull && sudo docker compose up -d`. Step by step: [Manual Install](https://github.com/AtGlance-system/atGlance/wiki/Manual-Install);
overlays: [Optional Features](https://github.com/AtGlance-system/atGlance/wiki/Optional-Features).

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | The console: app, worker, scheduler, MySQL, Redis, Kong API gateway (`:8002`), MCP server, controller |
| `docker-compose.domain.yml` | Optional: your own domain with HTTPS on ports 80/443 (built-in Caddy) — [docs/custom-domain.md](docs/custom-domain.md) |
| `docker-compose.nfs.yml` | Optional, premium: NFS exports mounted in the console — [docs/NFS_STORAGE.md](docs/NFS_STORAGE.md) |
| `.env.example` | Template for `.env` (the installer writes `.env` for you) |
| `scripts/atglance-installer.sh` / `.ps1` | First install, or upgrade, on Linux / Windows |
| `scripts/atglance_update.sh` / `.ps1` | Upgrade an installed console (backup, image pull, database check, rollback on failure) |
| `docs/` | Licensing, custom domain, API gateway, NFS, Azure Storage, storage targets, backups, workspace settings |

## Requirements

| | |
|---|---|
| Server | Linux (amd64) with Docker Engine and Compose, or Windows 10/11 / Windows Server with Docker Desktop |
| Size | 2 vCPU, 2 GB RAM, 5 GB free disk |
| Access | root or sudo (Linux), Administrator (Windows); outbound internet to ghcr.io and GitHub |
| Ports | **8000** web console, **8002** API gateway for the `atglance` CLI. Keep 3306 and 6379 closed |

## Back up

- **`APP_KEY`** in `storage/.env` inside the `app-storage` volume: it encrypts stored secrets. Keep a copy.
- **The database:** the updater makes one before each update.

Scheduled backups and restore: [Backup and Restore](https://github.com/AtGlance-system/atGlance/wiki/Backup-and-Restore). Problems: [Troubleshooting](https://github.com/AtGlance-system/atGlance/wiki/Troubleshooting).

Full guide, including Windows, container services (ECS, Kubernetes) and rollback:
**[INSTALLATION-UPDATE.md](INSTALLATION-UPDATE.md)** and the **[wiki](https://github.com/AtGlance-system/atGlance/wiki)**. What changed in each release: [CHANGELOG.md](CHANGELOG.md).
