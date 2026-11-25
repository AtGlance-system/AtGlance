# NFS Storage (premium)

NFS Storage is a **premium** plugin (licence key `nfs_storage`). It provides:
- registered NFS exports, with monitoring and alerts;
- up to **4 exports mounted on the console** at the same time, one per **mount slot**;
- a choice of **which export stores what**: database backups, config & files backups and the configuration
  files can each go to a different NFS server;
- NFS as a destination in **Backup & Restore**, and Local ↔ NFS in **Migration**.

It is included in a licence with the **Premium Features** bundle (every premium feature), or granted on its own
(`nfs_storage`) by a custom licence. Switch it on in **Site Settings › Plugins**. Once it is licensed and switched
on, the **NFS Storage** tab appears in Site Settings, right after S3 Configuration (super admin only).

Without the bundle or the `nfs_storage` grant in the licence:
- the Plugins card shows it as not licensed, and there is no NFS Storage tab;
- the API answers 403 `FEATURE_NOT_LICENSED`;
- the checks do not run.

See `docs/PLUGINS.md` and `docs/LICENSING.md`.

Mounting NFS exports on **managed nodes** is not part of this release. The console has no command channel to
the nodes: the `atglance` CLI only pushes data.

## How the console reaches the exports

### Automatic mounting (the controller)

With `docker-compose.nfs.yml`, the **controller** (`ce-atglance-controller`, `composer/bin/controller.php`) mounts
the registered exports by itself: no `.env` lines, no restart. It mounts an export at `/app/nfs/<slot>` in the
app, worker and scheduler containers **only when all of these hold** (`php artisan nfs:wanted`):

1. the licence includes NFS Storage;
2. NFS Storage is switched on in Plugins;
3. the export is registered (with a mount slot, picked automatically);
4. the export is **in use**: it keeps the configuration files, files are still recorded on it, a backup is set
   to save on it, or a migration to or from it is being analysed or run.

Anything else is unmounted: without the licence or with the plugin off nothing is mounted, and after the
configuration files are migrated off an export (and no backup uses it) it is unmounted. Files on the NFS
server are never deleted. A registered export that is not in use is only checked for reachability ("Not in
use, so not mounted"); it is mounted on demand when a backup or a migration needs it.

The controller re-checks every 10 seconds (mounts come back after a console restart), and at once when the
console calls `POST /nfs/apply` (register, remove, Check now, plugin switch, backup settings, migration). Its
report is `storage/app/nfs/mounter-status.json`; a mount error shows next to the export. The NFS connection is
made from the host network, so the export must allow the host's IP address.

Without the licence, configuration files kept on NFS are not reachable until the licence is back (they stay on
the NFS server). Move them to another storage before removing NFS Storage from the licence.

Security: `docker-compose.nfs.yml` gives the controller the rights mounting needs (privileged, host PID
namespace). It acts only on the console containers of its own stack (same storage volume) and only with
validated values; the console containers stay unprivileged. Without that file the controller never mounts
anything. Its other calls are unchanged (MCP start, stop, status; everything else 403).

### Docker NFS volumes (alternative)

The console never mounts anything itself. It runs as unprivileged containers, and mounting needs root
(`CAP_SYS_ADMIN`). The **Docker daemon** mounts the exports as NFS volumes, defined in
`docker-compose.nfs.yml`, in the app, worker and scheduler containers:

| Mount slot | Path in the containers | `.env` |
|---|---|---|
| 1 | `/app/nfs/1` | `NFS_1_SERVER`, `NFS_1_EXPORT`, `NFS_1_OPTIONS` |
| 2 | `/app/nfs/2` | `NFS_2_SERVER`, `NFS_2_EXPORT`, `NFS_2_OPTIONS` |
| 3 | `/app/nfs/3` | `NFS_3_SERVER`, `NFS_3_EXPORT`, `NFS_3_OPTIONS` |
| 4 | `/app/nfs/4` | `NFS_4_SERVER`, `NFS_4_EXPORT`, `NFS_4_OPTIONS` |

A slot without `NFS_<n>_SERVER` is an empty local volume. The console treats it as not mounted and never
writes to it.

1. In the install directory's `.env`, one block per export:

   ```
   NFS_1_SERVER=nfs1.example.internal
   NFS_1_EXPORT=/exports/backups
   NFS_1_OPTIONS=nfsvers=4.1,rw,hard,timeo=50,retrans=2

   NFS_2_SERVER=nfs2.example.internal
   NFS_2_EXPORT=/exports/configs
   ```

   `NFS_<n>_OPTIONS` is optional; the line above is its default.
2. Restart with the overlay. Add `-f docker-compose.domain.yml` if you use the built-in proxy:

   ```
   docker compose -f docker-compose.yml -f docker-compose.nfs.yml up -d -y
   ```

   Docker keeps the options a volume was created with. When a slot's server or export changes (or a slot
   gets its first server), `-y` lets Docker recreate that slot's volume. Without it, Compose asks first.
   Recreating the volume only removes Docker's mount definition; it never deletes files on the NFS server.
3. In **Site Settings › Storage › NFS**, register each export: a **name**, the **NFS server** (IP or host
   name) and the **export path**. That is all; the console picks a free mount slot, uses read-write and lets
   the client and server agree on the NFS version. Then click **Check now**. Each status must be `mounted`.
   If one is not, the page shows the `.env` lines to add for its slot.
4. Choose what is stored where (next section).

When all 4 slots are used, a new export is registered as monitor only (checked for reachability, not
mounted) until a slot is free.

The updater (`atglance_update.sh` / `.ps1`) delivers `docker-compose.nfs.yml` with each release and keeps the
compose files the console was started with.

### Requirements

- **Docker host:** the kernel NFS client (on Linux, `nfs-common` on Debian or Ubuntu, `nfs-utils` on RHEL).
  The console image itself includes `nfs-common`, for `showmount`.
- **Docker Compose:** a version whose `up` has `-y`. Without it, run `docker volume rm <project>_nfs-<n>`
  (after `docker compose down`) before `up -d` when a slot changes.
- **NFS server:** NFSv3, v4.1 or v4.2. TCP 2049 must be reachable from the Docker host; NFSv3 also needs
  rpcbind (111).
- **Server export:** must allow the Docker host's address with `rw`. Root squashing may be on; the console
  writes as root, which then maps to the anonymous user.
- **Advanced options** (validated, never free text): NFS version (Automatic, the default, or 3 / 4.1 / 4.2),
  security (below), `hard` or `soft`, `timeo` (1 to 6000 tenths of a second) and `retrans` (0 to 10).

  Use `hard` for backups, because `soft` can turn a slow server into corrupted writes. The console's own
  checks have their own timeouts, so a hard mount never hangs it.

### Security: IP allow-list or Kerberos

NFS has no user name or password. A share that asks for one is SMB (Windows file sharing), which the console
does not use.

- **IP allow-list** (the default, `sec=sys`): the NFS server allows the Docker host's IP address on the export.
  Most NAS devices, Linux file servers and cloud NFS shares (for example Azure Files NFS) work this way.
- **Kerberos** (`sec=krb5`, `krb5i` for integrity, `krb5p` for encryption): common with Active Directory. Choose
  it under Advanced › Security. The Docker host mounts the share, so the host must be a Kerberos client first:
  joined to the realm or domain (for example `realm join` or `adcli join`), with a keytab
  (`/etc/krb5.keytab`) and the `rpc-gssd` service running. The console cannot do this from the browser.
- **NFS over TLS** (certificates, Linux 6.5+ with `tlshd`) is not offered yet.

## What is stored where

Each kind of data is stored on one export. Only exports with a mount slot can be chosen.

| Data | Where it is chosen | Stored at |
|---|---|---|
| Database backups | NFS Storage › What is stored where, or Backup & Restore › Database › Save to NFS | `backups/database/YYYY/MM/` on the export |
| Config & files backups | NFS Storage › What is stored where, or Backup & Restore › Config & files › Save to NFS | `backups/config/YYYY/MM/` on the export |
| Configuration files | Migration › Move configuration files (To: the export), analysed then migrated | the same paths as on local storage |

- **Backups:** "Copies to keep" is applied on the chosen export. The restore list shows backups from every
  mounted export ("NFS: <name>"), also from an export no longer chosen, so older copies stay restorable.
- **Configuration files:** while they are kept on NFS, new uploads go to the same export. To move them
  anywhere else (Local, another export, S3, Azure), use the Migration tab (see
  [STORAGE_TARGETS.md](STORAGE_TARGETS.md)).
- An export that stores something cannot be removed: choose another export for it first, or migrate the
  configuration files back to Local.
- New kinds of data are added to the same list (`NfsTarget::USES`).

## What NFS Storage shows

| Status | Meaning |
|---|---|
| `mounted` | The export is mounted at its slot and answers. Capacity (used / free) and latency (time of a `stat` call) are recorded |
| `not_mounted` | Nothing, or another export, is mounted at the slot. The page shows the `.env` lines to add |
| `stale` | The mount did not answer within 5 seconds, or returned a stale file handle |
| `unreachable` | TCP 2049 on the server does not answer within 3 seconds |
| `reachable` | (monitor-only exports) The server answers |

- **Check frequency:** `nfs:monitor` runs every minute while licensed. **Check now** runs it on demand.
- **Status on the command line:** `php artisan nfs:status --json`.
- **Test** checks the connection to one server, and lists its exports when it supports `showmount`
  (NFSv3 servers; NFSv4-only servers do not list exports).

### Alerts

These are sent to the notification groups that subscribe to them:
- **NFS mount stale** (`nfs.stale`);
- **NFS server unreachable** (`nfs.unreachable`);
- **NFS export capacity above threshold** (`nfs.capacity_high`). The threshold is 90 % by default and can be
  set from 50 to 99 % on the page.

Alerts are sent when the status changes, not on every check. The capacity alert is sent once, and again only
after usage drops below the threshold and crosses it again.

### Audit

All of the following go to the activity log:
- registering, testing and removing exports;
- choosing what is stored where;
- settings changes;
- status changes;
- capacity alerts.

Removing an export's registration never unmounts anything.

## Workspace backups on NFS

An export is available at organization level and at workspace level. In its pane, a super admin
switches on **Allow workspaces to use this storage for their backups and restores**; workspace
admins can then choose it on their workspace's Backups tab. Workspaces use the organization's
registered export and mount slot (no mounts of their own) and only the folder
`backups/workspace-{id}-{name}/` on it. An export a workspace backs up to counts as in use, so the
controller mounts it. Without `nfs_storage` the export is locked for workspaces too: their
backups there are skipped (`Target not licensed`), and saving it returns 403. See
`docs/workspace-settings.md`.

## Downgrade (licence without `nfs_storage`)

When a new licence no longer includes `nfs_storage`:
- **kept:** the export registrations and the choices of what is stored where;
- **left in place:** the Docker NFS mounts;
- **left untouched:** files on the NFS servers;
- **still readable:** configuration files stored on NFS. Migration keeps offering **NFS to Local** so they can
  be moved back;
- **stopped:** the checks, NFS backups and restores from NFS; NFS actions answer 403;
- **banner** for the super admin: "NFS Storage license removed; existing mounts and backups left untouched."

## Security

- **Validation:** host names, IP addresses and export paths (absolute, without `..`, spaces or shell
  characters) are validated. Mount options are built only from validated fields. The mount slot is a number
  from 1 to 4.
- **Commands:** the console only runs `stat` and `showmount`, always as argument arrays and never through a
  shell, and always with a timeout.
- **No root needed:** the console only reads `/proc/mounts` and the mounted paths. Root is needed only by the
  Docker daemon, to mount the volumes.
- **Never into the container:** a slot's disk points at the slot only while `/proc/mounts` shows the
  registered export there (same server and path). Otherwise every write fails with an error, so data never
  lands silently inside the container.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `not_mounted`, "Nothing is mounted at /app/nfs/2" | Set `NFS_2_SERVER` and `NFS_2_EXPORT` in `.env` and start the console with `-f docker-compose.nfs.yml up -d -y` |
| `docker compose up` asks "Recreate (data will be lost)?" for a `nfs-<n>` volume | A slot's server or export changed. Answer yes, or use `up -d -y`. Files on the NFS server are not touched |
| `docker compose up` fails with "failed to mount local volume … connection refused" or "no route to host" | The Docker host cannot reach `NFS_<n>_SERVER:2049`. Check the server, the firewall and the address. On Docker Desktop, use an address the Docker VM can reach |
| "… mounted from host:/other" | The slot holds another export than the one registered. Fix `.env` or the registration so the slot number matches |
| `unreachable` | The server is down, or the firewall blocks TCP 2049 |
| `stale` | The server stopped answering while the export was mounted. It usually recovers when the server is back; otherwise restart the console containers |
| Permission denied when writing | Allow the Docker host in the server's export with `rw` (and `no_root_squash`, or give the anonymous user write access) |

## Local test environment

`dev/nfs/` provides an NFS server (NFS-Ganesha) and console containers: run `make nfs-up` / `make nfs-test` /
`make nfs-down`. See `dev/nfs/README.md`.
