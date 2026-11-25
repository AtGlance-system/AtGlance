# Storage targets

Backups, restore and the Site Settings > Storage Targets page work through **storage targets**. A target is
one place backups live: the console's local disk, S3, an NFS export, an Azure Blob Storage
container. The backup engine (`App\Services\BackupService`) and restore
(`App\Services\RestoreService`) know nothing about a target's kind; they work through the types
in `composer/app/Support/Storage/`.

## The pieces

| Type | Role |
|---|---|
| `StorageTarget` (interface) | One target: `key()`, `label()`, `featureKey()`, `testConnection()`, `put($name, $stream)` (atomic), `get()`, `list($prefix)`, `delete()`, `stat()`, `verify($name, $sha256)` |
| `DiskTarget` | A target backed by a Laravel filesystem disk (Flysystem). Local, S3, NFS and Azure are all disks |
| `BackupDestination` | One place a backup type may be copied to: selected or not, copies to keep, skip message (switched off, not licensed), how to prepare the target, and the wording of the backup log |
| `RestoreSource` | One place backups are listed from on the Restore page: a target with its folder prefixes, or an error to show |
| `StorageTargetProvider` (interface) | Supplies one kind of target: its backup destinations for a type, its restore sources, the restore source keys it owns, the configured target to restore from, and its restorable path pattern |
| `StorageTargets` | Registry: core first, then the storage plugins, in a fixed order. `overview()` and `usage($key)` feed the Storage Targets page |
| `TargetStatus` | One row of the Storage Targets page: kind, name, location, state (`ready`, `locked`, `off`, `unavailable`), message, uses, and the target when it can be listed |
| `RetentionLockedException` | Thrown by `delete()` when a retention rule still protects the object (Azure immutability). Pruning keeps the object and carries on |

Providers today:

| Provider | Targets | Feature | Restore source keys |
|---|---|---|---|
| `Support\Storage\CoreTargets` | S3, local disk | `s3_backup`, `backup_restore` (free) | `s3`, `local` |
| `Plugins\NfsStorage\Services\NfsTargets` | one per mounted export slot | `nfs_storage` (premium) | `nfs-1` ... `nfs-4` |
| `Plugins\AzureStorage\Services\AzureTargets` | one per Azure target | `azure_storage` (premium) | `azure-<id>` |
| `Plugins\GoogleDrive\Services\DriveTargets` | the connected Google Drive (organization backups only, never the configuration files) | `backup_restore` (free) | `gdrive` |

Google Drive is not a Flysystem disk: `DriveTarget` implements `StorageTarget` directly on the Drive
REST API and keeps each file's backup path in its `appProperties`. See [GOOGLE_DRIVE.md](GOOGLE_DRIVE.md).

## What every target gets for free

The engine does the rest, the same way for every target:

- the archive (database dump, configuration and console files), named
  `backups/{type}/YYYY/MM/{type}-backup-YYYYMMDD-HHMMSS.*`;
- retention: the newest *n* copies per target are kept, only files named like backups are
  counted;
- the backup log line and the `backup.succeeded` / `backup.failed` notifications;
- the Restore list (newest first, from every source), the pre-restore snapshot, the restore.

## Site Settings > Storage

One tab for every storage: a left pane with **Overview**, **Local**, **S3**, **NFS** and **Azure**
(each with its state: Saving configs, Active, Not licensed, Off, Needs attention), and on the right
the storage's own settings (S3 keys, NFS exports, Azure targets). The old S3 Configuration, NFS
Storage, Azure Storage and Storage Targets tabs open the matching pane (`?tab=storage&store=...`).

- **Overview** shows `StorageTargets::overview()`: every configured target with its state and
  uses. When the tab opens, the page asks `GET /admin/settings/storage-targets/usage?key=<key>`
  for each target that can be listed (files under `backups/`, size, newest backup).
- Each storage has **Keep future configuration files in this storage?** (see below).
- Each storage except Local has **Allow workspaces to use this storage for their backups and
  restores** (super admin only), see [Workspace backups](#workspace-backups).
- **Workspaces** lists every workspace with its backup destinations, schedule, encryption and
  last backup, with filters for workspace, storage and result.

## Workspace backups

Every storage is available at organization level and at workspace level. Workspaces do not
configure storages of their own: a super admin allows an organization storage for workspaces
(`App\Support\WorkspaceStorage`, admin setting `workspace_storage_allowed`, a JSON list of
storage keys; unset means `["s3"]`, as before), and each workspace then chooses it on its
Backups tab. A workspace only reads and writes `backups/workspace-{id}-{name}/` on it
(`BackupService::workspacePrefix()`); paths are checked against that prefix and the backup file
name pattern, so `..` or another workspace's path is refused (404).

`WorkspaceStorage::target($key)` gives the ready target or the reason it cannot be used: not
allowed, not licensed (`Target not licensed: ...`), switched off, unavailable. An NFS export a
workspace backs up to counts as in use, so the controller mounts it
(`NfsMounter::inUse()`). Licence rules are the same as for the organization: NFS and Azure are
locked without their feature, S3 needs none. See `docs/workspace-settings.md` for the Backups
tab, retention, encryption and restore.

## Where configuration files are kept

`App\Support\ConfigStorage`. The same rules for S3, NFS and Azure:

- **One storage at a time**: Local, S3, one NFS export or one Azure target. It is changed only
  by a migration (Migration tab, below).
- **Choosing a storage moves every file there** (queued job `MoveConfigFiles`): each file is
  copied, read back and its SHA-256 compared, its record is pointed at the new storage, and only
  then the old copy is removed. Choosing Local moves everything back and removes the remote
  copies the same way. Progress shows on the Storage and Migration tabs.
- **New uploads** are saved on the console first, then the queued job `PushConfigFileToStorage`
  moves them to the remote storage the same way (5 tries, the worker's backoff). Until then the
  file is served from the console. No configuration file stays on the console once it is on the
  remote storage.
- **Nothing is lost when a remote is unreachable**: a file whose copy fails stays where it is
  (its record says where) and `config-storage:sync` (every 5 minutes, or *Sync now*) moves it
  later.
- **Reads** use the storage on the file's record, then the chosen storage, then the console.
- Turning S3 on does not move files by itself; choose S3 to do that.
- Consoles that never chose keep the earlier rules: where the last migration put the files (NFS,
  Azure), else S3 while S3 is enabled, else Local.
- Files on NFS or Azure stay readable without the licence; a storage that is not licensed or
  switched off cannot be chosen.

## Migration tab

At the top, the storage that keeps the configuration files now (also on the Storage tab). On the
left, every storage with its state (Current, Enabled, Not licensed, Switched off, Unavailable).
On the right, **Move configuration files** (`App\Support\ConfigMigration`):

1. *From* is preset to the current storage; *To* lists Local and every other enabled storage.
2. **Analyse** (a few seconds, nothing changes): counts the files to move and their size from the
   database, times copying up to three of them to the destination and back, and estimates the
   migration time from that speed.
3. **Start migration** (only after an analysis of that destination, valid 60 minutes): every file is
   copied, read back and compared, its record pointed at the destination, and the old copy removed.
   Progress and the time left update live.
4. Only when every file has moved is the destination saved as the storage for future files; files
   uploaded meanwhile are moved in a last pass. If any file cannot be moved, the storage is not
   changed (files already moved stay readable); analyse again to retry.

While an analysis or a migration runs, the storage cannot be changed. Both run as queued jobs in
slices of 40 seconds, each queuing the next, so they fit the worker's timeout. The Storage tab's
*Move configuration files here* opens this form with the destination chosen.

**Move to a new server** is greyed out for now (turned on with `ATGLANCE_CONSOLE_MIGRATION=true`).
The older direction-based endpoints (`/admin/settings/migration/*`) still work but are not shown.

## Licensing

A provider is always registered, licensed or not, so a backup set to a premium target never
disappears silently:

- while the feature is not licensed (or the plugin is switched off), the destination is skipped
  with a message (Azure: "Target not licensed: ..."), its settings are kept and nothing is
  written or deleted;
- its restore sources are not listed, and a restore from it is refused;
- routes and pages of the plugin answer 403 `FEATURE_NOT_LICENSED` (PluginManager).

## Adding a target (for example SFTP or Google Cloud Storage)

1. Add the feature to `contracts/fixtures/features.json` (tier, minimum console version) and copy
   it to the console and site catalogs (`contracts/FEATURES.md`).
2. Create the plugin in `composer/app/Plugins/<Name>/` with a `plugin.php` (see
   PLUGINS.md): its settings model and migration, a Site Settings tab, routes.
3. Give it a Flysystem adapter (or reuse one) and a disk name per target, e.g. `sftp-<id>`.
   `put()` must be atomic: write under a temporary name and rename, or use an upload that only
   becomes visible when committed.
4. Write the provider (`implements StorageTargetProvider`):
   - `backupDestinations()`: read the type's settings (add the keys to `BackupSettings::get()` /
     `save()`), return one `BackupDestination` with `skipMessage` set while unlicensed;
   - `restoreSources()`: one `RestoreSource::target()` per usable target, `RestoreSource::error()`
     for one that cannot be used now;
   - `owns()` / `sourceKeys()` / `restoreTarget()` for restore;
   - `overview()`: one `TargetStatus` per configured target, also the locked and unusable ones,
     without any network call (usage is listed later, per target).
5. Add the provider class to `StorageTargets::BUILT_IN` (or call `StorageTargets::register()`).
6. Add the destination to the Backup & Restore form (`admin/partials/backup-section.blade.php`) and
   its validation (`AdminDashboardController::updateBackupRestoreSettings()`).
7. Tests: a fake of the remote service (see `tests/Support/FakeAzureBlob.php`), the gating
   matrix, and a dev environment with an emulator if one exists (see `dev/azure/`).
