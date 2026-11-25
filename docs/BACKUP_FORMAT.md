# Backup file format

Every backup the console writes (scheduled, Run now, migration) is the archive itself, optionally
encrypted, plus a manifest next to it. The same format is used on every storage target (local,
S3, NFS, Azure). How to use backups: [scheduled-backups.md](scheduled-backups.md). Targets:
[STORAGE_TARGETS.md](STORAGE_TARGETS.md).

## Files

| File | Content |
|---|---|
| `database-backup-YYYYMMDD-HHMMSS.sql.gz` | SQL dump of every table, gzipped |
| `config-backup-YYYYMMDD-HHMMSS.zip` | `config.json` (configuration file records and contents, systems, services, AI validations) and `files/` (console files) |
| `migration-backup-YYYYMMDD-HHMMSS.zip` | `database.sql.gz`, `config.zip`, `console.json` (Installation ID, `APP_KEY`, version, address) and `licence/` (licence files). Always encrypted |
| `<file>.enc` | The file above, encrypted (below) |
| `<file>.manifest.json` | The manifest of the stored file (below). Written after the file, so a backup with a manifest is complete |

## Manifest

```json
{
    "format": 1,
    "type": "database",
    "name": "database-backup-20261028-020000.sql.gz.enc",
    "created_at": "2026-10-28T02:00:00+00:00",
    "console_version": "2.4.0",
    "install_id": "6f1c...",
    "size": 1048621,
    "sha256": "<SHA-256 of the stored file>",
    "plain_size": 1048576,
    "plain_sha256": "<SHA-256 before encryption>",
    "encryption": { "algorithm": "AES-256-GCM, Argon2id", "format": 1 },
    "migration": false
}
```

`encryption` is `null` for a plain backup. A console refuses a manifest with a higher `format`
than it knows. The manifest holds no secret. Code: `App\Support\Backup\Manifest`.

Before a restore (and a copy), the downloaded file is checked against `size` and `sha256`; after
decryption, against `plain_sha256`. A mismatch stops the restore before anything changes.

## Encryption

`App\Support\Backup\BackupCrypto`. The key is derived from the passphrase with Argon2id
(libsodium `crypto_pwhash`, default cost "interactive": 2 passes, 64 MiB), with a random 16-byte
salt per file. The data is encrypted with AES-256-GCM in chunks of 1 MiB, so files of any size
are streamed.

```
header  "ATGLENC1" | version (1 byte) | opslimit (4, big endian) | memlimit (4, BE)
        | salt (16) | nonce prefix (8) | chunk size (4, BE)               = 45 bytes
chunk   final flag (1) | ciphertext length (4, BE) | ciphertext | GCM tag (16)
```

- Nonce of chunk *i*: the 8-byte random prefix followed by *i* (4 bytes, big endian).
- Additional data of chunk *i*: the whole header, *i*, and the final flag. Changing the header,
  reordering or dropping chunks, or cutting the file short makes decryption fail.
- The last chunk has final flag 1; data after it is refused.
- A wrong passphrase and a changed file give the same error: "The backup could not be decrypted:
  wrong passphrase, or the file was changed or damaged." A half-decrypted file is deleted.

To decrypt outside the console, derive the key with any Argon2id13 implementation using the
header's salt, opslimit and memlimit (32-byte output), then decrypt each chunk as above.
