# Changelog

## Unreleased

### Removed: Add servers from the console

- **Add server** (Remote Deploy, premium key `remote_deploy`) is removed: the Add server button and dialog,
  Update CLI / Redeploy / Restart agent / Enable agent over a saved login, the deployment accounts page, the
  `add_servers` workspace permission, the `deploy.failed` event and the `atglance-deployer` helper in the image.
- Registered servers stay, including the ones added with Add server: they keep syncing with their
  deployment service account key, which admins now manage in the Service Accounts list.
- Add a server on the server itself: `sudo atglance login`, then `sudo atglance system register`
  (and `sudo atglance agent install` for live monitoring).
- The licence key `remote_deploy` is retired (`contracts/FEATURES.md`); licences that list it are not affected.
- The tables `deploy_targets` and `deploy_runs` are left in place on existing installs (no schema change).

### Agent Monitoring: retention, old servers, updates

- **Data retention** in Settings > Agents (super admin): seven settings for raw metrics, 5-minute averages, heartbeats, check results, outages and alerts, events and sessions, and request output. Each shows its size and oldest record. **Clean up now** runs the cleanup at once.
- The daily `agents:retention` (03:40) now also cleans the MySQL history. Servers, the latest tables, open outages and open alerts are never deleted.
- **Remove old servers** on the Servers page (super admin): remove one offline server, or all servers offline for more than N days. Connected servers are never removed. Registered systems and configuration backups are kept.
- **Update agents** from the Servers page: **Update** and **Update all outdated** for connected agents 3.0.0 or newer, one update at a time per server. The release list is read from `ATGLANCE_CLI_RELEASES_URL` (`config/cli_releases.php`).
- **Servers** is in the top bar; Servers, Service heartbeat and Requests are no longer in the sidebar. See [docs/AGENT_MONITORING.md](docs/AGENT_MONITORING.md).

### Google Drive backups (free)

- New **Google Drive** plugin: keep copies of the Database and Configuration & console files backups in your own Google Drive, and restore from them. Free, part of Backup & Restore. See [docs/GOOGLE_DRIVE.md](docs/GOOGLE_DRIVE.md).
- You use your own Google OAuth client:
  - **Web application** clients connect through the console's callback (HTTPS or localhost).
  - **Desktop app** clients work on headless consoles: open the Google link on any device and paste the address back.
- The console asks for the `drive.file` scope only, so it sees only the files it created. Backups go to `My Drive/AtGlance-Backups/<folder>/`.
- Google Drive is a storage target like S3, NFS and Azure. Schedules, copies to keep, encryption, manifests, checksum checks, pre-restore snapshots and rollback all work the same way.
- The client secret and the refresh token are stored encrypted with `APP_KEY`.
- `gdrive:health` runs daily and sends two new organization notifications:
  - `gdrive.reconnect_needed` when Google no longer accepts the access;
  - `gdrive.quota_low` when the Drive is almost full.

### Other changes

- Muted text is darker across the console and readable (`--ag-muted` #646b75).
- Status badges use darker text colors.
- The dashboard's Vulnerabilities tile no longer shows zero counts in red.

## 2.3.0 (unreleased)

### Security advisory: PHP2Shell (CVE-2026-109618, CVE-2026-109619)

**Severity: critical.** These are two unauthenticated remote code execution flaws in the PHP runtime, also called "Black October":
- A single multipart POST is enough. PHP parses it before the console's own code (Laravel) runs, so login and API keys do not protect against it.
- One flaw needs a memory-limit failure while PHP parses the form.
- The other abuses session upload progress, which PHP turns on by default.

Every AtGlance console image up to and including 2.2.0 ran on the floating `php:8.2-cli` base image. Whether an image is affected depends on the PHP version it was built with. PHP 8.2.35, 8.3.36, 8.4.27 and 8.5.12 fix the flaws.

**What changed in 2.3.0:**
- **Patched runtime:**
  - the console image is pinned to `php:8.2.35-cli`;
  - the build fails on any older PHP and prints `php -v` in the build log;
  - the updater warns when a running console has PHP older than 8.2.35.
- **PHP hardening** (`docker/php/zz-atglance.ini`):
  - session upload progress is turned off;
  - `memory_limit` is 256M;
  - `post_max_size` is 12M and `upload_max_filesize` is 10M;
  - `max_file_uploads` is 5 and `max_input_vars` is 1000;
  - `expose_php` is Off.
- **Request size limit at the edge:**
  - Caddy and the Kong gateway refuse request bodies over 12 MB with `413`, before PHP parses them. Config uploads stay limited to 10 MB.
  - Kong also rate limits the config upload routes to 60 requests per minute per IP: `/files/upload`, `/config-files/upload` and `POST /api/v1/config-files`.
- **The PHP server is no longer exposed:**
  - Host port 8000 is now served by the built-in Caddy proxy in the app container. The URL `http://<host>:8000` is unchanged.
  - The PHP development server (`php artisan serve`) is reached only through Caddy and, inside the Docker network, through the gateway. The CLI keeps using port 8002, and port 8000 for logout.
- **Pinned release:** `docker-compose.yml` and the Helm chart default to `2.3.0` instead of `latest`.

**What you must do:**
1. **Update now.** Run the updater, or re-run the installer, which runs it. At the end the updater must show `PHP 8.2.35 (PHP2Shell fixed)` or newer.
   - On Linux, as root:

     ```bash
     curl -fsSL https://raw.githubusercontent.com/niketchandra/atGlance-managementGUI/main/scripts/atglance_update.sh | sudo bash -s -- --yes
     ```

   - On Windows, in an administrator PowerShell:

     ```powershell
     irm https://raw.githubusercontent.com/niketchandra/atGlance-managementGUI/main/scripts/atglance_update.ps1 -OutFile atglance_update.ps1
     powershell -ExecutionPolicy Bypass -File .\atglance_update.ps1
     ```

   See [INSTALLATION-UPDATE.md](INSTALLATION-UPDATE.md#updating) for details.
2. **If your console was reachable from the internet before you updated**, assume it may have been compromised. Rotate:
   - every API key (`atgla-...`) and service account key (`atgsa-...`), in the console under Settings and Service Accounts;
   - the database passwords: `DB_PASSWORD` and `DB_ROOT_PASSWORD` in the install directory's `.env`. Change them in MySQL first, then in `.env`, then run `docker compose up -d`;
   - the Redis password (`REDIS_PASSWORD`);
   - the SMTP, S3 and SSO secrets stored in the console.

   Review the activity log and the security events for unknown sign-ins or changes.
3. Consoles on Kubernetes or a container service that send traffic straight to container port 8000 should point their load balancer at container port **8080** (Caddy). They should also set a 12 MB request body limit on their ingress or load balancer.

### Add servers from the console (premium)

- **Add server** on Systems Registered installs the atglance CLI on a Linux, Windows or macOS server over SSH or WinRM (HTTPS only) and registers it. Premium feature `remote_deploy`. See [docs/REMOTE_DEPLOY.md](docs/REMOTE_DEPLOY.md).
- Who: the super admin always. Workspace admins need the new **Add servers from the console** permission, which only the super admin grants on the workspace's Members tab.
- **Deployment accounts**: a separate kind of service account for Add server, managed by the super admin. Their keys are never shown. Rotating a key updates servers with saved credentials, and the old key works for 24 more hours.
- The host key (SSH) or certificate (WinRM HTTPS) is shown and confirmed before any password is sent. A changed key is refused until the super admin accepts it.
- Logins are used once, unless **Save these credentials for later** is ticked. They are encrypted with `APP_KEY`. Saved logins power **Update CLI**, **Redeploy** and key rotation on the system's row.
- A failed deploy gets one **Retry**. If that fails too, the console undoes its changes on the server. The organization event `deploy.failed` ("Server deploy failed") reports it.
- A CLI already registered with this console on an older version is not deployed over: update it on the server.
- Needs CLI 3.0 (`atglance system enroll`, `atglance version --json`), internet access from the console to atglance.live (`ATGLANCE_CLI_RELEASES_URL`) and the queue worker. macOS is offered once a release has a macOS file.
- If the licence is removed, deployed servers keep syncing, and the Add server actions are locked.

### Other changes in 2.3.0

- Licence files (`.atgl`): offline activation with a licence key or a licence file. See [docs/LICENSING.md](docs/LICENSING.md).
- System keys are stored only as hashes. A migration clears the plaintext copies that are left.
