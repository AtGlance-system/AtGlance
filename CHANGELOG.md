# Changelog

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
     curl -fsSL https://raw.githubusercontent.com/AtGlance-system/atGlance/main/scripts/atglance_update.sh | sudo bash -s -- --yes
     ```

   - On Windows, in an administrator PowerShell:

     ```powershell
     irm https://raw.githubusercontent.com/AtGlance-system/atGlance/main/scripts/atglance_update.ps1 -OutFile atglance_update.ps1
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

### Other changes in 2.3.0

- Licence files (`.atgl`): offline activation with a licence key or a licence file. See [docs/LICENSING.md](docs/LICENSING.md).
- System keys are stored only as hashes. A migration clears the plaintext copies that are left.
