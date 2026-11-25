# Google Drive backups

The **Google Drive** plugin keeps copies of the console's backups in your own Google Drive and
restores from them. It is free (part of Backup & Restore, feature `backup_restore`) and needs no
licence entry.

Google Drive only stores backup files at points in time. The console never reads anything on the
Drive as live configuration, and the configuration files the atglance CLI uploads are never kept
there (Site Settings > Storage does not offer "Keep future configuration files" for Google Drive).

What is backed up is the same as for every other storage: the **Database backup** and the
**Configuration & console files backup** (see [scheduled-backups.md](scheduled-backups.md)).
Files on the hosts themselves (nginx, PHP, DNS, firewall, cron, SSH keys) are not: the console
cannot reach them. The configuration files the CLI collects from the hosts are in the
configuration backup.

## Before you start

- A Google account whose Drive will hold the backups. A Google Workspace account works too.
- A Google Cloud project you can manage (free).
- The console reaches `accounts.google.com`, `oauth2.googleapis.com` and `www.googleapis.com` over
  HTTPS (port 443).
- Super admin (`rbac_id` 100) in the console.

## 1. Create the Google OAuth client

The console uses **your own** OAuth client. Google shows its name on the consent screen, and the
console asks only for the `drive.file` scope: it can see and change only the files it created,
nothing else in the Drive.

In the [Google Cloud console](https://console.cloud.google.com/):

1. Create a project (or choose one), for example `atglance-backups`.
2. **APIs & Services > Library**: search for **Google Drive API** and click **Enable**.
3. **APIs & Services > OAuth consent screen** (Google Auth Platform > Branding / Audience / Data access):
   1. App name, for example `AtGlance backups`, and your support email.
   2. Audience: **Internal** (Google Workspace, only your organization's accounts) or **External**.
   3. Data access: add the scope `.../auth/drive.file`. It is a non-sensitive scope, so Google does
      not need to review the app.
   4. For **External**: **Publish app** (status "In production"). While the app is in "Testing",
      Google ends the access after **7 days** and the console shows **Needs reconnect**.
4. **APIs & Services > Credentials > Create credentials > OAuth client ID**. Choose the type:

   | Your console | Client type | Redirect URI to add |
   |---|---|---|
   | Opens over HTTPS (custom domain, see [custom-domain.md](custom-domain.md)), or on `localhost` | **Web application** | The address shown in the console under *Web application*, `https://<console>/admin/google-drive/callback` |
   | No HTTPS, or a server without a browser | **Desktop app** | None |

   Google accepts web redirects only to HTTPS addresses or localhost, so a console on
   `http://<ip>:9000` uses a Desktop app client.
5. Copy the **Client ID** (ends with `.apps.googleusercontent.com`) and the **Client secret**
   (starts with `GOCSPX-`).

## 2. Switch the plugin on

Site Settings > **Plugins** > **Google Drive** > switch it on. A **Google Drive** pane appears in
Site Settings > **Storage**.

## 3. Save the client and connect

Site Settings > Storage > **Google Drive**:

1. **Your Google OAuth client**: choose the client type, paste the client ID and secret, and check
   the **Folder in Google Drive** (default: the console's host name). Backups go to
   `My Drive/AtGlance-Backups/<folder>/`. Give every console its own folder name.
   **Save OAuth client**. The secret is stored encrypted with `APP_KEY` and never shown again.
2. **Connect Google Drive**:
   - **Web application**: the browser goes to Google. Sign in with the account whose Drive keeps the
     backups and click **Allow**. Google sends you back to the console.
   - **Desktop app** (headless console): the console shows a Google link. Open it on any device
     with a browser (your laptop or phone), sign in and click **Allow**. The browser then tries to
     open `http://127.0.0.1/?state=...&code=...` and shows an error: that is expected. Copy the
     whole address from the address bar (or only the `code` value) and paste it in **Code or address
     from Google** within 15 minutes, then **Finish connecting**.
3. The **Connection** card shows the account, the folder and the storage used.
   **Test connection** writes, reads and deletes a small probe file in the folder.
   **Reconnect** starts the same consent again (after "Needs reconnect", or to switch accounts).

A pasted address must come from the redirect of the request you started: `127.0.0.1` or
`localhost` (any port) for a Desktop app client, the console's callback host for a Web
application client. The consent link asks for `drive.file` only, with
`include_granted_scopes=false`, so no scope granted to the client earlier is added.

The connection is protected with PKCE and a one-time `state` value. The refresh token is stored
encrypted with `APP_KEY`; the short-lived access token is cached encrypted. Neither reaches the
browser or a log.

## 4. Choose Google Drive for the backups

Site Settings > **Backup & Restore**: in the Database and the Configuration & console files
sections, tick **Upload to Google Drive** and set **Copies to keep** (default 7, up to 365).
Save. The schedule, **Run now**, retention, manifests and notifications work as for the other
storages ([scheduled-backups.md](scheduled-backups.md)).

**Turn on Encryption** (Backup & Restore > Encryption). The backups hold secrets: password
hashes, settings, and the built-in proxy's CA private key. With encryption on, Google Drive only
ever holds encrypted files (AES-256-GCM, key derived from your passphrase with Argon2id).
Keep the passphrase somewhere safe: an encrypted backup cannot be read without it, and a restore
on another console asks for it.

## Restore

The Restore list of each Backup & Restore section shows the backups in Google Drive with the
location `Google Drive: AtGlance-Backups/<folder>`, next to the local, S3, NFS and Azure ones.
A restore from Google Drive works like any other: password, then the file is downloaded and
checked against Google's MD5 and the manifest's SHA-256, decrypted, a snapshot of the current
state is saved, and a failed restore is rolled back. See
[scheduled-backups.md](scheduled-backups.md#restore).

To restore on a **new console** (the old one is gone): install it, switch the plugin on, save the
**same OAuth client** and the **same folder name**, connect with the same Google account, and
restore from the list. The `drive.file` scope only shows files created by that OAuth client, so a
different client does not see the old backups.

## How files are kept in the Drive

- One folder per console: `My Drive/AtGlance-Backups/<folder>/`, created when missing (also when
  someone deleted it).
- Each backup and its manifest are one file each, under their own names
  (`database-backup-20261112-020000.sql.gz.enc`, `....manifest.json`). The full backup path is
  stored in the file's private properties (`appProperties.atglance_path`), so the paths are the same
  as on every other storage.
- Uploads are resumable and sent in 8 MB chunks. After a failed chunk the console asks Google how
  much arrived and goes on from there; 429, 5xx and rate-limit errors are retried with exponential
  backoff. A file appears in the Drive only once complete.
- Retention deletes the oldest backups (and their manifests) beyond **Copies to keep** for good,
  not to the trash. Files you put in the folder yourself are ignored and never deleted.

## Health check and notifications

`php artisan gdrive:health` runs every day at 05:20 (the scheduler container). It checks that
Google still accepts the console's access and how much room is left. It sends, once each:

| Event | When |
|---|---|
| `gdrive.reconnect_needed` | Google refuses the refresh token: revoked in the Google Account, password changed, client secret reset, or the app in "Testing" for over 7 days. Also sent when this happens during a backup or a test. |
| `gdrive.quota_low` | Less than 10% of the Drive is free, or less than twice the newest backup. Sent again only after there was room in between. |

Subscribe organization notification groups to them on the Notifications page or the
Backup & Restore notifications card. Both are also in the activity log.

## Disconnect

**Disconnect** needs your password. It revokes the access at Google and forgets the refresh token
and the account. The backups in Google Drive and the OAuth client settings are kept. Saving a
different OAuth client also ends the connection.

Switching the plugin off (Plugins) keeps the connection: backups set to Google Drive are skipped
("switched off") and Google Drive is not listed for restore until it is on again.

## Troubleshooting

| Message | What to do |
|---|---|
| `redirect_uri_mismatch` | Add the exact callback address shown in the pane to the Web application client, or use a Desktop app client. |
| Google only redirects to HTTPS addresses | Open the console over HTTPS ([custom-domain.md](custom-domain.md)) or use a Desktop app client. |
| `accessNotConfigured` / Enable the Google Drive API | Enable the Google Drive API in the client's Google Cloud project. |
| Google did not return a refresh token | Remove the app's access at <https://myaccount.google.com/permissions>, then click Reconnect. |
| That address is not from Google's redirect | You pasted the address of another page. Paste the address of the page Google opened after Allow (`http://127.0.0.1/...` for a Desktop app client). |
| Needs reconnect after 7 days | Publish the consent screen (In production), then connect again. |
| The Google Drive is full | Free up space, buy more storage, or keep fewer copies. |
| The connection request expired | More than 15 minutes passed after Connect. Click Connect (or Reconnect) again. |

## Code

| Part | File |
|---|---|
| Plugin manifest | `composer/app/Plugins/GoogleDrive/plugin.php` |
| OAuth (consent URL, PKCE, exchange, refresh, revoke, pasted codes) | `Services/DriveOAuth.php` |
| Drive REST client (resumable upload, list, download, delete, quota) | `Services/DriveClient.php` |
| Settings, connection, tokens, quota warning | `Services/GoogleDrive.php` |
| Storage target and provider | `Services/DriveTarget.php`, `Services/DriveTargets.php` |
| Pages | `Http/Controllers/DriveController.php`, `views/tab.blade.php` |
| Health check | `Console/DriveHealthCommand.php` |
| Tests (fake Google on `Http::fake`) | `composer/tests/Feature/GoogleDrive*Test.php`, `composer/tests/Support/FakeGoogleDrive.php` |
