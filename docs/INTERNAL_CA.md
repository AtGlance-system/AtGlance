# Remote Tunnel internal CA

Remote Tunnel has its own small certificate authority (`App\Plugins\RemoteTunnel\Services\InternalCa`). It
issues the certificates that connectors and the tunnel service use for mutual TLS. Nothing else trusts it.

The SSH CA and the connector certificates are active. The IPsec gateway and Point-to-Site device
certificates belong to parked features: their code is kept, but nothing issues them while IPsec and
Point-to-Site are parked (PARKED(remote-tunnel-simplify), [PARKED_FEATURES.md](PARKED_FEATURES.md)).

## Hierarchy

| Certificate | Key | Lifetime | Use |
|---|---|---|---|
| Root | ECDSA P-384 | 10 years | Signs the intermediate only. Its SHA-256 fingerprint is part of every enrollment token. |
| Intermediate | ECDSA P-256 | 2 years | Signs connector and tunnel service certificates |
| Connector | ECDSA P-256, made on the connector | 24 hours to 7 days (profile setting, default 24 hours) | Client certificate, CN = connector UUID |
| IPsec gateway (per profile, PARKED) | ECDSA P-256 | 1 year, renewed 30 days before expiry | Server/client auth and the IKE intermediate EKU, subject and SAN = the console IKE ID |
| Tunnel service | ECDSA P-256 | 1 year | Server certificate for `TUNNEL_PUBLIC_NAMES`. The bootstrap hands it out with the chain (leaf + intermediate + root) and caches it for 30 days. |

The CA is created on first use (`InternalCa::ensure`). Keys are stored in `tunnel_ca`, encrypted with the
app key (`Crypt`).

**Backups.** Database backups contain the CA keys, still encrypted with `APP_KEY`, like every other secret
of the console. A restore on the same console (same `APP_KEY`) brings the CA back, and connectors keep
working. Turn on backup encryption (Backup & Restore → Encryption) for a second layer. Anyone with both an
unencrypted backup and the `APP_KEY` could sign connector certificates. Every issued
certificate is recorded in `tunnel_certificates` (serial, kind, subject, validity, revocation).

## Signing rules

The CA signs a CSR only when all of these are true:
- its key is EC P-256 or P-384;
- the signature verifies;
- the CN is the expected connector UUID;
- the request comes through an enrollment or renewal that the app accepted.

The CA ignores any extensions in the CSR. It sets key usage and extended key usage itself (client auth for
connectors, server auth for the tunnel service).

## Revocation

`InternalCa::revoke($serial, $reason)` marks the certificate revoked. `InternalCa::denyList()` returns the
revoked serials that have not expired yet. The tunnel service fetches the list on every bootstrap (30 s) and
refuses those serials during the handshake. Renewal revokes the previous serial with the reason
"superseded".

## Rotation

`InternalCa::rotate()` makes a new intermediate, signed by the current root. Existing connector
certificates stay valid until they expire: connectors present their own chain, and the trust bundle holds
the roots. `rotate(newRoot: true)` also makes a new root. That changes the fingerprint, and every connector
must then be enrolled again. Phase 1 has no page or command for rotation. Call it from `php artisan tinker`.

Administrators see the CA status (certificates, expiry, fingerprint) on the Remote Tunnel page.
