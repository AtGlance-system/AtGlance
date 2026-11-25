# AtGlance Management Console: deploy and upgrade

This repository holds only what you need to **install** the AtGlance Management Console (Community
Edition) on a server, or to **upgrade** an existing one: the Docker Compose files, the installer
and updater scripts, and the guides. The console itself ships as Docker images on Docker Hub
(`atglance/ce-atglance-*`); no source code is needed.

Every free feature is included. Premium features (NFS Storage, Azure Storage, Remote Tunnel, Agent
Monitoring) use the same images and switch on with a licence that includes them. Their optional
containers come from the overlay files below.

## Quick start

**Linux (Docker host), new install or upgrade:**

```bash
curl -fsSL https://raw.githubusercontent.com/AtGlance-system/AtGlance/main/scripts/atglance-installer.sh | sudo bash
```

**Windows (PowerShell, as Administrator):**

```powershell
irm https://raw.githubusercontent.com/AtGlance-system/AtGlance/main/scripts/atglance-installer.ps1 | iex
```

The installer checks the host (CPU, memory, Docker, Docker Compose), then runs the updater:
- **new server:** it creates `/opt/atglance` with a `.env` holding random passwords and starts the
  console;
- **existing console:** it backs up the database, updates the images and compose files, checks the
  database, and keeps data and passwords.

Then open `http://<server>:8000`, run the setup wizard and add the licence (Settings > Licence).

**Upgrade later** on a running console:

```bash
curl -fsSL https://raw.githubusercontent.com/AtGlance-system/AtGlance/main/scripts/atglance_update.sh | sudo bash
```

Full guide, including manual Compose installs, Windows, container services (ECS) and rollback:
**[INSTALLATION-UPDATE.md](INSTALLATION-UPDATE.md)**. What changed in each release:
[CHANGELOG.md](CHANGELOG.md).

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | The console: app, worker, scheduler, MySQL, Redis, Kong API gateway (`:8002`), MCP server, controller |
| `docker-compose.domain.yml` | Optional: your own domain with HTTPS on ports 80/443 (built-in Caddy) — [docs/custom-domain.md](docs/custom-domain.md) |
| `docker-compose.nfs.yml` | Optional, premium: NFS exports mounted in the console — [docs/NFS_STORAGE.md](docs/NFS_STORAGE.md) |
| `docker-compose.agents.yml` | Optional, premium: Agent Monitoring (`agent-monitoring` service, agent gateway, agent worker, TimescaleDB) — [docs/AGENT_MONITORING.md](docs/AGENT_MONITORING.md) |
| `docker-compose.tunnel.yml` | Optional, premium: Remote Tunnel service — [docs/REMOTE_TUNNEL.md](docs/REMOTE_TUNNEL.md) |
| `docker-compose.tunnel-connectors.yml` | Optional, premium: publishes the Agent Tunnel connector port — [docs/AGENT_TUNNEL.md](docs/AGENT_TUNNEL.md) |
| `.env.example` | Template for `.env` (the installer writes `.env` for you) |
| `scripts/atglance-installer.sh` / `.ps1` | First install, or update, on Linux / Windows |
| `scripts/atglance_update.sh` / `.ps1` | Update an installed console (backup, image pull, database check, rollback on failure) |
| `docs/` | Guides for the optional parts: licensing, custom domain, API gateway, NFS, Azure Storage, Google Drive, backups, Agent Monitoring, Remote Tunnel |

## Manual install with Docker Compose

```bash
mkdir -p /opt/atglance && cd /opt/atglance
# copy docker-compose*.yml and .env.example here, then:
cp .env.example .env    # set DB_PASSWORD, DB_ROOT_PASSWORD and REDIS_PASSWORD (long random values)
docker compose up -d
```

Add overlays with `-f`, for example Agent Monitoring:

```bash
docker compose -f docker-compose.yml -f docker-compose.agents.yml up -d
```

The overlays need their own `.env` values (listed at the top of each file, for example
`ATGLANCE_AGENT_GATEWAY_SECRET` and `TSDB_PASSWORD` for Agent Monitoring). The updater keeps
whichever compose files a console was started with.

## Back up

- **`APP_KEY`** in `storage/.env` inside the `app-storage` volume: it encrypts stored secrets.
- **The database:** the updater makes one before each update.

See "Back up the app key" in [INSTALLATION-UPDATE.md](INSTALLATION-UPDATE.md).
