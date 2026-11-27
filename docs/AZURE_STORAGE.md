# Azure Storage (premium)

The Azure Storage plugin saves the console's scheduled and manual backups (database, and
configuration & console files) to **Azure Blob Storage** containers, and restores from them.
It is a premium feature, licence key `azure_storage`: included in the Premium Features bundle,
or granted on its own in a custom licence.

- The licence is checked **offline**, as for every feature. Azure Storage itself needs the
  console to reach Azure over HTTPS: the public endpoint, a **private endpoint**, or Azure Stack
  Hub. Consoles without that network access cannot use it.
- Backups to Azure use the same archive, names, retention and Restore list as local, S3 and NFS
  copies ([STORAGE_TARGETS.md](STORAGE_TARGETS.md), [scheduled-backups.md](scheduled-backups.md)).

## Set up

1. **Licence and switch.** With a licence that includes Azure Storage, turn it on in
   Site Settings > Plugins > Azure Storage. The Site Settings > Azure Storage tab appears.
2. **Container.** Your Azure administrator creates the container (the console never creates one:
   that is your organisation's decision), e.g. `atglance-backups`.
   One container per console, or one prefix per console in a shared container.
3. **Identity and role.** Give the identity the console signs in with the role
   **Storage Blob Data Contributor**, scoped to the container (not the account).
4. **Add the target** in Site Settings > Storage > Azure, then **Test connection**. The test signs
   in, checks the container, and writes, reads and deletes a
   probe blob `.atglance-probe-<random>`.
5. **Choose it** in Backup & Restore: tick *Upload to Azure* for the database and/or the
   configuration backup, pick the target and the number of copies to keep.
6. **Optional: keep the configuration files in Azure.** In Site Settings > Storage > Azure, answer
   *Keep future configuration files in this storage?* with **Yes, keep them here**. Every file is
   moved to Azure (copied, checked, then removed from the console); new uploads wait on the
   console only until they are copied. While Azure keeps the files, the plugin cannot be switched
   off and the target cannot be removed. Choosing Local moves them back.

Backups to Azure use the shared engine: with encryption on (Backup & Restore > Encryption) the
console encrypts each backup before it is uploaded, so the container only holds ciphertext; each
backup has a manifest with its checksum; **Copy to…** copies a backup between Azure and another
target; and a migration backup saved in Azure can be given to a new console as a SAS link
([scheduled-backups.md](scheduled-backups.md)).

Site Settings > Storage > Overview lists every target (local, S3, NFS exports, Azure targets) with
its state, what it is used for, and the backups kept on it (count, size, newest).

## Settings of a target

| Field | Notes |
|---|---|
| Storage account name | 3 to 24 lowercase letters and digits |
| Container | 3 to 63 lowercase letters, digits and single hyphens |
| Path prefix | Optional folder inside the container, e.g. `console-a`. Backups go to `<prefix>/backups/...` |
| Cloud / endpoint | Azure public cloud (`*.blob.core.windows.net`), Azure Government (`*.blob.core.usgovcloudapi.net`), Azure China (`*.blob.core.chinacloudapi.cn`), or a custom Blob endpoint (private endpoint, Azure Stack, Azurite) |
| Sign in with | See below |
| Access tier | Hot, Cool or Cold. **Archive is refused**: an archived blob must be rehydrated (up to 15 hours) before it can be restored |
| Block size, parallel uploads | Upload tuning, default 8 MB blocks, 4 in parallel |
| HTTP(S) proxy | For consoles with restricted egress, e.g. `http://proxy.internal:3128` (credentials allowed, stored encrypted) |

## Authentication methods

In order of preference:

| Method | When | What to enter |
|---|---|---|
| **Managed identity** (recommended in Azure) | The console runs on an Azure VM, scale set, AKS, App Service or Container Apps | Nothing, or the client ID of a user-assigned identity. Tokens come from the Instance Metadata Service (`169.254.169.254`) or `IDENTITY_ENDPOINT` |
| **Service principal** (recommended elsewhere) | Any console with access to Entra ID | Tenant ID, client (application) ID, and a client secret **or** a certificate (PEM with the certificate and its unencrypted private key; sent as a signed client assertion). The token authority follows the cloud: `login.microsoftonline.com`, `login.microsoftonline.us`, `login.chinacloudapi.cn` |
| SAS token | When no identity can be used | A container-scoped SAS with read, add, create, write, delete and list. The console reads its expiry (`se=`) and warns from 14 days before it |
| Account key / connection string (least preferred) | Emulators, or as a last resort | The key, or the connection string (its `BlobEndpoint` overrides the cloud setting). The console warns: the key gives full access to the whole storage account |

### Least-privilege example (Azure CLI)

```bash
# Container
az storage container create --account-name mystorageacct --name atglance-backups --auth-mode login

# Service principal with access to this container only
SCOPE=$(az storage account show -n mystorageacct --query id -o tsv)/blobServices/default/containers/atglance-backups
az ad sp create-for-rbac --name atglance-console-backups --role "Storage Blob Data Contributor" --scopes "$SCOPE"
# -> appId (client ID), password (client secret), tenant (tenant ID)
```

For a managed identity, assign the same role to the identity:

```bash
az role assignment create --assignee <principal-id> --role "Storage Blob Data Contributor" --scope "$SCOPE"
```

## How uploads work

- The archive is uploaded as a block blob: blocks of the configured size, several in parallel,
  each with its MD5. The blob becomes visible only when the block list is committed, so an
  interrupted upload never leaves a backup that looks valid (Azure discards uncommitted blocks
  after 7 days).
- The MD5 of the whole blob is stored (`Content-MD5`) and checked again on every download.
- Every request is retried up to 6 times with exponential backoff and jitter on connection
  errors and HTTP 408, 429, 500, 502, 503 and 504; `Retry-After` is honoured (throttling).
  Failed blocks are re-sent alone, not the whole archive.
- The archive is built on the console's disk first, then streamed to Azure: keep free space in
  the app-storage volume of about the size of the largest backup. Streaming without a local copy
  comes with the encrypted backup format (later release).
- Azure's own encryption at rest always applies. Client-side encryption of the archive with a
  passphrase is planned for the encrypted backup format.

## Immutability, soft delete, versioning

- With a **time-based retention policy or legal hold** on the container, Test connection says so.
  When older copies are pruned, a copy still under retention is kept and logged
  ("cannot be deleted yet"); the backup run does not fail.
- Soft delete and versioning work as configured in Azure: a pruned copy stays recoverable in
  Azure for the soft-delete period.

## Secrets

- The client secret, certificate, SAS token, account key, connection string and proxy URL are
  stored encrypted with the console's `APP_KEY` (the same mechanism as the S3 and SMTP secrets)
  and are never sent back to the browser: edit forms show "saved (leave empty to keep)".
- Secrets are removed from logs, errors, the activity log and API answers (SAS signatures,
  `AccountKey=`, Shared Key and bearer tokens, client secrets and assertions, proxy passwords).
- Back up `APP_KEY` (see [scheduled-backups.md](scheduled-backups.md)): without it the saved
  Azure secrets cannot be decrypted after a restore on a new server.

## Workspace backups on Azure

A target is available at organization level and at workspace level. In its pane, a super admin
switches on **Allow workspaces to use this storage for their backups and restores**; workspace
admins can then choose it on their workspace's Backups tab. Workspaces never see the target's
credentials and only use `backups/workspace-{id}-{name}/` in the container (under the target's path
prefix). Uploads, manifests, retention and immutability work as for the organization backups.
Without `azure_storage` the target is locked for workspaces too: their backups there are skipped
(`Target not licensed`), and saving it returns 403. See `docs/workspace-settings.md`.

## Without the licence (downgrade)

When a new licence no longer includes `azure_storage`:

- scheduled backups to Azure are skipped with "Target not licensed"; other destinations of the
  same backup still run;
- the targets, their settings and every blob in Azure stay untouched;
- restoring from Azure is refused on this console, and the Azure tab and API answer 403
  `FEATURE_NOT_LICENSED`;
- configuration files stored in Azure (Migration) stay readable and new uploads still go there,
  so no data becomes unreachable; *Azure to Local* stays offered in Migration so they can be
  moved back;
- Storage Targets shows the targets as *Not licensed* and does not list them;
- a banner at the top of the console explains it (while Azure targets exist).

Uploading a licence that includes the feature again brings everything back as it was.

## Status API

`GET /api/v1/azure/status` (API key of a super admin, licensed): the targets with their location,
cloud, sign-in method, tier, last test result and warnings. No secret is returned.

## Troubleshooting

| Message | Cause and fix |
|---|---|
| `HTTP 403 AuthorizationPermissionMismatch` | The identity has no data role. Assign **Storage Blob Data Contributor** on the container |
| `HTTP 403 AuthenticationFailed` / `AuthorizationFailure` | Wrong key, expired or wrong-scope SAS token, or the clock of the console is far off |
| `The container ... does not exist` | Ask your Azure administrator to create it (Storage account > Data storage > Containers > + Container), then test again |
| `Could not get an Azure token for the managed identity` | The console does not run in Azure, or no identity is assigned to its host |
| `Could not get an Azure token for the service principal: AADSTS...` | Wrong tenant, client ID or secret, or the secret expired |
| `Could not reach Azure Storage` | No network path: firewall, proxy, private endpoint DNS. Set the proxy in the target |
| `does not match its stored MD5 checksum` | The blob was changed or damaged in Azure. Use another copy |
| A copy is "kept: cannot be deleted yet" | Immutability policy or legal hold on the container; expected |

Local testing with the Azurite emulator: `dev/azure/README.md` in the console source.
