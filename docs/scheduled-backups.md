# Scheduled Backups

The **Backup & Restore** tab in `/admin/settings` has two separate sections:

| Section | What it backs up | Command | Who can restore |
|---|---|---|---|
| **Database backup & restore** | A gzipped SQL dump of every table (made in PHP, so `mysqldump` is not needed). | `php artisan backup:database` | Super admin only (`rbac_id` 100) |
| **Configuration & console files backup & restore** | A zip with `config.json` (the `services`, `system_register`, `configuration_files`, `raw_data` and `config_ai_validations` tables; `raw_data.file_data` holds each file's content) and `files/` (the console's own files, see below). | `php artisan backup:config` | Admins (`rbac_id` 100 and 101) |

`php artisan backup:portal` still exists and runs the database backup, so older
host crontabs keep working.

Each workspace can also back up its own stored config files on its own schedule,
with notes (`php artisan backup:workspace {id}`). See `docs/workspace-settings.md`.

The logic is in `composer/app/Services/BackupService.php`, the settings in
`composer/app/Support/BackupSettings.php`, and the commands in
`composer/routes/console.php`.

## Console files in the configuration backup

`BackupService::CONSOLE_FILE_ROOTS` lists them:

| Archive folder | Directory | Contents |
|---|---|---|
| `files/public/` | `storage/app/public` | Branding images and other public uploads |
| `files/caddy/pki/` | `storage/caddy/pki` | The built-in proxy's local CA (including its private key) |
| `files/caddy/certificates/` | `storage/caddy/certificates` | Certificates the built-in proxy issued |

Keep the S3 bucket private: the archive contains the CA private key.

## Destinations

Every destination is a *storage target* (local, S3, NFS export, Azure container) behind one
interface; see [STORAGE_TARGETS.md](STORAGE_TARGETS.md).

## Settings for each section

| Setting | Values |
|---|---|
| Scheduled backup | On / Off |
| How often | Every hour, 6 hours, 12 hours, day, week, month, or a **custom cron expression** (5 fields: minute hour day month weekday, for example `30 2 * * *`) |
| Keep local copies on this server | Yes / No, and how many copies to keep (1-365) |
| Upload to S3 | Yes / No, and how many copies to keep (1-365). Needs S3 enabled on the S3 tab. |
| Save to NFS | Yes / No, the export, and how many copies to keep. NFS Storage plugin (premium `nfs_storage`), see [NFS_STORAGE.md](NFS_STORAGE.md). |
| Upload to Azure | Yes / No, the Azure target, and how many copies to keep. Azure Storage plugin (premium `azure_storage`), see [AZURE_STORAGE.md](AZURE_STORAGE.md). |
| Upload to Google Drive | Yes / No, and how many copies to keep. Google Drive plugin (free), see [GOOGLE_DRIVE.md](GOOGLE_DRIVE.md). |

At least one destination is required when the schedule is on. The master
**Enable Backup and Restore** switch turns every backup off; restores still work.

Settings are stored in `admin_settings` as `backup_{database|config}_{enabled|frequency|cron_expression|to_s3|to_local|keep_s3|keep_local|to_nfs|keep_nfs|to_azure|keep_azure|azure_target}`.
Settings saved before the split (`backup_config_to_s3`, `backup_config_cron`,
`backup_portal_to_s3`, `backup_portal_cron`) are read as defaults (S3 only)
until the tab is saved again.

Saving with the files backup set to S3 also sets
`configuration_file_base_location` and `CONFIGURATION_FILES_BASE_DISK` to `s3`,
as before.

## Where backups are kept

| Place | Path |
|---|---|
| S3 | `backups/{database|config}/YYYY/MM/{database-backup-YYYYMMDD-HHMMSS.sql.gz | config-backup-YYYYMMDD-HHMMSS.zip}` |
| Local copies | The same path on the local disk: `storage/app/private/backups/...` |
| NFS | The same path on the chosen export (`/app/nfs/<slot>/backups/...`) |
| Azure | The same path in the target's container, under its optional path prefix (`<prefix>/backups/...`) |
| Google Drive | One folder, `My Drive/AtGlance-Backups/<folder>/`, each file under its own name; the same path is kept in the file's `appProperties` |
| Manifests | Next to every backup: `<backup file>.manifest.json` (see [BACKUP_FORMAT.md](BACKUP_FORMAT.md)) |
| Encrypted backups | The same names with `.enc` added, e.g. `database-backup-YYYYMMDD-HHMMSS.sql.gz.enc` |
| Migration backups | `backups/migration/YYYY/MM/migration-backup-YYYYMMDD-HHMMSS.zip.enc` on the target chosen when it was made |
| Pre-restore snapshots | `storage/app/private/backups/snapshots/{type}/pre-restore-...` |
| Workspace backups | `backups/workspace-{id}-{name}/YYYY/MM/workspace-backup-YYYYMMDD-HHMMSS.zip` (`.zip.enc` when encrypted) on local and the organization storages allowed for workspaces (S3, NFS, Azure); see [workspace-settings.md](workspace-settings.md#backups) |

Local copies are on the same server and volume as the app. They help with
mistakes and bad restores, not with losing the server; use S3 for that.

## Retention

After each successful backup, the oldest copies beyond the "copies to keep"
number are deleted, separately for each destination (local, S3, NFS, Azure). In an Azure
container with an immutability policy, a copy still under retention is kept and logged;
the run does not fail. Only files named like this
service's backups (`database-backup-*.sql.gz`, `config-backup-*.zip` and the
older `config-backup-*.json.gz`, each also with `.enc`) are counted and deleted, together with
their manifests. Pre-restore snapshots and migration backups are never deleted automatically.
Workspace backups (`backups/workspace-{id}-{name}/`) are not counted here: each workspace keeps its own
number of copies and its retention only deletes that workspace's `workspace-backup-*` files.

## Run now

Each section has a **Run now** button (`POST /admin/settings/backups/run`,
6 per minute). It uses the saved destinations and retention even when the
schedule is off, but not when the master switch is off.

## When a run is skipped

A run is recorded as `skipped` and nothing is written when:

- Backup & Restore (`backup_restore_enabled`) is off
- The section's schedule is off (scheduled runs only)
- No destination is selected
- S3 is the only destination and S3 is disabled
- NFS or Azure is the only destination and its plugin is off or not licensed
  (Azure: "Target not licensed")
- This console was marked migrated after a migration backup (scheduled runs only; see
  [Moving a console to a new server](#moving-a-console-to-a-new-server))

When both destinations are selected and S3 is disabled, the local copy is
still saved and the message says that no S3 copy was made.

## Last run status

Each run writes its result to `admin_settings` under `backup_database_last_run`
or `backup_config_last_run`: `status` (`success`, `failed` or `skipped`),
`message`, `object_key`, `started_at` and `finished_at`. The section shows it
under its buttons, and the **Crons** tab lists each scheduled backup with its
destinations.

A failed run writes a `Scheduled backup failed` entry to the Laravel log, sends
the `backup.failed` notification, and the command exits with code 1.

## Running the scheduler in deployment

The scheduler must run in exactly one place. Pick one of these options.

**Docker Compose:** the `scheduler` service in `docker-compose.yml` runs
`php artisan schedule:work`.

**Cron on the host:** add this entry to the crontab of the user that owns the
app:

```cron
* * * * * cd /path/to/composer && php artisan schedule:run >> /dev/null 2>&1
```

The scheduler reads the settings from `admin_settings` each time it evaluates
the schedule, so a change in the UI applies without a restart. Schedules are
registered only after the installer has finished.

The scheduler container must share these with the app container:

- The same `APP_KEY`. The S3 secret in `admin_settings` is encrypted with it.
- The `storage/` volume. Local copies, snapshots, console files and the installer marker live there.

## Encryption and manifests

Backup & Restore > **Encryption** (super admin) turns on client-side encryption: every new
backup is encrypted on the console with AES-256-GCM before it is written anywhere, with a key
derived from a passphrase (Argon2id). S3, NFS and Azure only ever hold the encrypted file. The
passphrase is stored encrypted with `APP_KEY`, so this console restores its own backups without
asking. A restore on another console, or of a backup made before the passphrase was changed,
asks for the passphrase. Keep it somewhere safe: an encrypted backup cannot be read without it.

Every backup, encrypted or not, gets a manifest next to it with its type, the console version and
Installation ID, and the SHA-256 of the file before and after encryption. A restore checks the
downloaded file against the manifest before it changes anything. Backups made before manifests
existed have none and still restore.

File format and manifest fields: [BACKUP_FORMAT.md](BACKUP_FORMAT.md).

## Restore

Each section has its own **Restore** list (`GET /admin/settings/backups?section=database|files`).
It loads when the tab is opened and shows, newest first:

- backups in S3 (when S3 is enabled)
- backups on every mounted NFS export and in every Azure target (while licensed and switched on)
- local copies on this server
- pre-restore snapshots on this server

To restore, pick a backup, confirm your password, and tick the overwrite
confirmation.

| Backup | What a restore does |
|---|---|
| Database | Replaces the whole database with the dump, then runs pending migrations. Everyone may need to sign in again. |
| Older portal backup (`portal-backup-*.zip`, listed in the Database section) | Replaces the database with its `database.sql`, runs migrations, and writes its `.env` back. Restart the app containers afterwards. |
| Configuration & console files | Adds missing records back and resets changed records to their backed-up values; records created after the backup are kept. Writes each configuration file's content back to its disk (S3 when available, otherwise local), and writes the console files back. Older `.json.gz` backups (no console files) can still be restored. |

Every restore:

- Needs the current user's password (and, for an encrypted backup from another console or an
  older passphrase, the backup passphrase).
- Checks the file first: its checksum against the manifest, and decryption. A changed or damaged
  file, or a wrong passphrase, stops the restore before anything changes.
- Saves a snapshot of the current state, as a local backup of the
  same type. The success or error message names the snapshot. To undo a
  restore that worked, restore that snapshot.
- If the restore fails part-way, puts the snapshot back at once (rollback) and says so. Only if
  the rollback fails too does the message ask you to restore the snapshot by hand.
- A database restore replaces the whole database: tables the dump does not have (added by a
  newer version) are dropped and created again by the migrations.
- Writes an entry to `activity_logs` with the backup, the snapshot, and the
  result.

The code is in `composer/app/Services/RestoreService.php` and
`composer/app/Http/Controllers/BackupRestoreController.php`.

## Copy a backup to another target

**Copy to…** next to a backup in a Restore list copies it, with its manifest, to another target
that is ready now (local, S3, an NFS export, an Azure target), under the same path
(`POST /admin/settings/backups/copy`). The copy is read back and its SHA-256 compared before the
copy is reported as done; the original is not changed. Use it to keep an off-site copy of an
important backup, or to put a backup where a new console can read it.

## Moving a console to a new server

Greyed out for now (Migration > Move to a new server): turn it on with
`ATGLANCE_CONSOLE_MIGRATION=true` in the app `.env` once it is released.

Migration > **Move to a new server** (super admin, password confirmed) makes
a **migration backup**: one encrypted file with the database dump, the configuration & console
files, the licence files, the Installation ID and `APP_KEY`. It is always encrypted with the
passphrase entered there (at least 12 characters, not saved), and saved on the target you choose,
with a download link.

On the new server:

1. Install the console as usual (`scripts/atglance-installer.sh`), then open it in a browser.
2. In the installer, choose **Restore / migrate an existing console**.
3. Give the migration backup: upload it, give an HTTPS link to it (an Azure SAS URL or an S3
   pre-signed URL), or the path of a file on the server (under `storage/app/backups/`, for
   example copied to `storage/app/backups/import/`, or on a mounted NFS export).
4. Enter the passphrase and this console's IP address, confirm, and restore.
5. Restart the console once (`docker compose restart` in its folder). Until then every page
   says so: the running processes read `APP_KEY` when they start, and only after the restart
   do they use the old console's key, so the saved secrets and the licence can be read.

The new console checks and decrypts the file before it touches anything (a wrong passphrase
installs nothing), then takes over the old console's `APP_KEY` (saved secrets stay readable),
database, configuration file contents, licence and Installation ID (the licence keeps working;
after the restart in step 5), and you sign in with your existing account. Large migration backups: an upload is
limited by PHP's upload size; use a link or a server path for those.

Agents keep their API keys and systems. Point each agent at the new console's address (the
atglance CLI's console host), and it carries on as the same system.

Tick **Mark this console migrated** when making the migration backup to stop the old console's
scheduled backups (a banner says so), so the two consoles do not write next to each other's
backups. **Keep using this console** undoes it.

## Checking the schedule

```bash
php artisan schedule:list      # shows registered backups and next due time
php artisan backup:database    # runs the database backup now (scheduled settings)
php artisan backup:config
```

## Tests

`tests/Feature/ScheduledBackupTest.php` and `tests/Feature/BackupRestoreTest.php`
cover both archive formats, local copies and retention, S3 and local together,
custom cron, settings validation, Run now, the per-section restore lists,
database, configuration and older portal restores, and console file restores.
