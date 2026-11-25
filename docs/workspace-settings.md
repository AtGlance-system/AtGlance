# Workspace Settings

Each workspace has its own settings page. Open it from **Workspace** in the sidebar
(`/admin/workspaces/{id}`) or, as super admin, from the Enterprise Console
(`/workspace/{id}`).

## Who can change them

Any user can be a workspace admin (`workspace_user.is_admin = 1`), whatever their account role.

- **Workspace admins with the Admin role** (`rbac_id` 101) and **the super admin** (100) can change everything.
- **Workspace admins with the User role** (102) can change only what an Admin-role workspace admin (or the super admin) allows. This access is chosen in the Add Workspace Admin form, and later through **Edit** in the Access column of the Members table. It is stored in `workspace_user.permissions`.

| Access (`Workspace::PERMISSIONS`) | Default for a User-role workspace admin |
|---|---|
| `general`: change General settings (name, description) | Yes |
| `members`: add and remove users | Yes |
| `admins`: add and remove other workspace admins | No |
| `vulnerability_checks`: change Vulnerability Checks, run a check | No |
| `backups`: change Backups, run a backup | No |
| `notifications`: change Notifications and the workspace's channel groups | No |

- Workspace Tags can be changed by every workspace admin.
- Tabs without access are shown greyed out, read only.
- Only Admin-role workspace admins and the super admin can set access. A User-role workspace admin cannot change anyone's access, including their own.
- `Workspace::allows($user, $permission)` is the check, and the server returns 403 without it.
- Workspace status and deleting a workspace stay super-admin only.

Removing members:
- Nobody can remove themselves or the super admin.
- Removing a workspace admin needs `admins`; removing a user needs `members`.
- A workspace admin with the User role cannot remove a workspace admin who has the Admin role.
- See `AdminDashboardController::canRemoveMember()`.

## Tabs

| Tab | What it sets |
|---|---|
| General | Name and description. Super admin also sees status and Delete. |
| Members | Add workspace admins and users; one table of members with workspace role and account role. |
| Workspace Tags | Key/value tags (`env` = `prod`), added and removed by any workspace admin. Shown and filterable on Manage Workspaces and in the Enterprise Console. When editing a system in the workspace they are offered as `key=value`, and they are suggested in the Systems tag filter. `system_register.tags` is still a free comma-separated string. |
| Vulnerability Checks | Automatic AI review of the **latest version** of each config file: on upload, and/or on a schedule. See below. |
| Backups | Scheduled backup of the workspace's stored config files, with notes and history. See below. |
| Notifications | Which events the workspace sends, default member emails, channel groups, member choices. See `docs/notifications.md`. |
| Recent Activity | Read-only timeline of everything in the workspace. See below. |

## Recent Activity

`?tab=activity` on the workspace page, for anyone who can open it. Newest first, grouped by day, 50 per page
(`activity_page`), with filters: **Show** (`activity_type`: systems & services, config backups, AI reviews &
checks, workspace backups, members & settings), **System** (`activity_system`) and **Period** (`activity_period`:
24 hours, 7/30/90 days, all time; default 30 days). Each entry links to the system, file or AI review when there is one.

Built by `App\Support\WorkspaceActivity` from the records of the workspace's systems (`system_register.workspace_id`):

| Entry | Source |
|---|---|
| System registered | `system_register.created_at` |
| Service added | `services` |
| Config backed up (file, service, version) | `configuration_files` |
| AI review (result, trigger, model) | `config_ai_validations` |
| Vulnerability Check run (reviewed / failed / skipped) | `workspace_ai_runs` |
| Workspace backup (finished / failed / skipped) | `workspace_backup_runs` |
| System deregistered or reactivated, failed AI review, AI review deleted, queue reset, members added or removed, workspace settings changed, notification groups added / updated / deleted | `activity_logs` rows tagged with `workspace_id` / `system_register_id` (migration `2026_10_07_000000_add_workspace_to_activity_logs`; `ActivityRecorder::record(..., $workspaceId, $systemId)`) |

The migration matches past "Deregistered / Reactivated system" log entries to their system by owner and name.
Settings and member changes made before it are not in the timeline. Request payloads, secrets and notification
targets are never shown. The feed is only built when the tab is opened.

## Storage

| Table | Holds |
|---|---|
| `workspace_settings` | One JSON document per workspace (`App\Support\WorkspaceSettings`, defaults in `DEFAULTS`). `tags` is a list of `{key, value}`; older plain tags are read as keys with no value. |
| `workspace_backup_runs` | One row per workspace backup run: status, message, file path, storage key (`disk`: local, s3, nfs-<slot>, azure-<id>), `encrypted`, notes, who ran it. |
| `workspace_notification_preferences` | A member's own choice of events and email on/off for one workspace. |

`config_ai_validations.user_id` is nullable (automatic reviews have no user) and
`config_ai_validations.trigger` is `manual`, `upload` or `schedule`.

## Vulnerability Checks

- **Check new uploads**: after the CLI uploads a config (`POST /api/config-files/upload`),
  `App\Jobs\ValidateConfigWithAi` is queued for it.
- **Scheduled re-check**: `php artisan ai:validate-workspace {id}` runs on the chosen schedule
  (preset or custom five-field cron). It queues the latest version of every
  `(system, file name)` pair in the workspace. With **Skip files already reviewed**
  (the default), latest versions that already have a review are left out.
- **Run check now** runs the same sweep immediately.
- **Check Queue** on the tab shows the latest run as a progress bar with a percentage and counts of reviewed, skipped and failed files. It refreshes every 3 seconds while the run is in progress (`GET /admin/workspaces/{id}/ai/progress`). Each run is a `workspace_ai_runs` row; every queued job adds 1 to `done`, `skipped` or `failed` when it ends. A job killed by the worker counts as failed, so a run always finishes.

The job runs on the queue worker, with one try and a 200-second timeout, so a failing provider is not called again by retries.
It does nothing when AI Connect is off, or when a newer upload of the same file
arrived before it ran. A review with status `error` or `warning` sends
`ai.issues_found`. This also applies to manual reviews.

Every review is an AI call that your provider bills for. A re-check with skipping
turned off reviews every latest version on every run.

## Backups

A workspace backup holds the configuration files already stored in the console for
the workspace's systems. It is written to `workspace-backup-YYYYMMDD-HHMMSS.zip`
(`.zip.enc` when encrypted) under `backups/workspace-{id}-{name}/YYYY/MM/`, with a
`.manifest.json` next to it (checksum, size, `scope: workspace`, `workspace_id`).
The zip has:

- `config.json`: the workspace's rows of `system_register`, `services`,
  `configuration_files`, `raw_data` (file content) and `config_ai_validations`
- `workspace.json`: id, name, notes, time, counts
- `notes.txt`: the notes from the Backups tab, when set

The folder carries the workspace's name as a slug, e.g. `backups/workspace-3-ansible-prod/`, so the
workspace is easy to find in the bucket, export or container. The id keeps the folder unique (two
workspaces with one name). After a rename, new backups go to a folder with the new name; the old
folders (and `backups/workspace-{id}/` from before names were added) are still listed, restored and
counted for retention. The folders used are remembered in `backup_folders`.

### Where it is saved

**Save backups on** lists the destinations the workspace can use, each with its own
"copies to keep":

- **This console (local disk)**: always available.
- **Organization storages allowed for workspaces**: S3, NFS exports and Azure targets
  that a super admin allowed in Site Settings &rarr; Storage ("Allow workspaces to use
  this storage for their backups and restores"). S3 is allowed until the super admin
  turns it off, so workspaces that already used S3 keep working. NFS and Azure are not
  allowed until switched on.

Workspaces never get their own credentials or mounts. They use the organization's
bucket, exports and containers, and only their own folder `backups/workspace-{id}-{name}/`
in each. A destination shows its state: a lock badge when the licence does not
include it (NFS needs `nfs_storage`, Azure needs `azure_storage`; S3 needs no premium
feature), or Off / Unavailable with the reason. Saving a destination that is not
allowed fails validation; saving an unlicensed one returns 403. When the organization
stops allowing a storage, the workspace's backups there are kept but not written,
listed or restored, and the history says why ("... skipped: The organization does
not allow workspaces to use this storage.").

Settings saved before this version (`backup_to_local` / `backup_to_s3` with their
keep counts) are read as destinations until the workspace saves the tab again.

### Encryption

**Encrypt backups with a passphrase** encrypts each backup with AES-256-GCM (key
from the passphrase with Argon2id, the same format as the organization backups,
`docs/BACKUP_FORMAT.md`). The passphrase needs 12 characters or more and must be
typed twice. It is stored encrypted with `APP_KEY`, never logged, and cannot be
shown again. Leave the fields empty to keep the current one.

### Running

`php artisan backup:workspace {id}` runs one backup now; **Run backup now** does the
same. The schedule queues `App\Jobs\RunWorkspaceBackup`: at most
`workspace_backup_parallel` workspace backups (admin setting, default 2, 1 to 10)
run at the same time; the others wait a minute and try again. One workspace never
runs twice at once. After each successful copy, the oldest copies beyond "copies to
keep" are deleted on that destination, counting only this workspace's
`workspace-backup-*` files: other workspaces' and the organization's backups are
never touched.

The history lists the last 20 runs with their notes, where they were saved, and a
download link. A run sends `workspace.backup_succeeded` or `workspace.backup_failed`.

### Restore

**Restore &rarr; Find backups** lists this workspace's backups on every destination
it can use (`GET /admin/workspaces/{id}/backups/stored`). Each can be downloaded;
users with the Backups permission can restore it into **this workspace only**:

1. Your password, the backup passphrase when it is encrypted, and the
   "current data will be overwritten" confirmation.
2. The file is downloaded and checked against its manifest (checksum).
3. A backup whose `config.json` names another workspace is refused, even when it
   was copied into this workspace's folder.
4. The workspace goes into maintenance: CLI uploads for its systems get
   `503` with `Retry-After: 60` (at most 120 minutes, if a restore is interrupted).
5. A snapshot of the workspace is saved to
   `storage/app/private/backups/snapshots/workspace-{id}/pre-restore-...` (local disk).
6. The backup's rows of this workspace are written back (missing rows inserted,
   changed rows reset) and the file contents are written to where each file is kept.
   Systems and files that now belong to another workspace are skipped and counted.
   Rows added after the backup was made are kept.
7. When a step fails, the snapshot is put back.

Restores are logged as `workspace.backup_restored` / `workspace.backup_restore_failed` and sent as
`workspace.restore_succeeded` / `workspace.restore_failed` (a failure also goes to the organization's
groups as `organization.workspace_restore_failed`). The **Notifications** card on the Backups tab
chooses which notification groups get the backup and restore events (`docs/notifications.md`).
The organization configuration backup (`docs/scheduled-backups.md`) still covers
every workspace and can be restored as before.

### Permissions

Super admins manage every workspace of their organization; workspace admins with
the Backups permission manage their own. Other members can see the tab (read only)
and list and download backups, but cannot change settings or restore. A workspace
you do not belong to answers 404, so its existence does not leak.

## Schedules

Both schedules are registered in `composer/routes/console.php` from
`WorkspaceSettings::workspacesWith(...)`. They only run for active workspaces.
The scheduler container (`schedule:work`) re-reads them every minute, so changes
apply without a restart. Check them with `php artisan schedule:list`.
