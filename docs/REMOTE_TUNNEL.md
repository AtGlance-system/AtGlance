# Remote Tunnel

Premium plugin (licence feature `remote_tunnel`, console 3.0.0 or later): secure, audited access to
servers on private networks — web terminal, file transfer (SFTP), web page and TCP port forwarding, and
console features that need to reach a server.

**AtGlance provides secure, audited access to servers. Your organization provides network connectivity.**
Organizations already connect their networks (site-to-site VPN, MPLS, SD-WAN, ExpressRoute/Direct Connect,
peering, the company's remote-access VPN), set up by their network team. Remote Tunnel uses that
connectivity; it does not rebuild it. Where no route exists and the network team will not add one, a small
outbound-only connector in the remote network opens audited sessions to listed servers.

The feature is included in the **Premium Features** bundle and can also be granted on its own in a custom
licence (docs/LICENSING.md).

- Code: `composer/app/Plugins/RemoteTunnel/` (plugin) and `tunnel/` (Go: tunnel service and connector).
- Deploy: one optional container, `ce-atglance-tunnel` (`docker-compose.tunnel.yml`). Agent Tunnel
  connectors also need `docker-compose.tunnel-connectors.yml`, which publishes the connector port.
- Parked (code kept, off by default): Site-to-Site IPsec, Point-to-Site VPN, relay mode, subnet router
  (NAT, Site DNS, policy matrix) and Site-to-Site transit — see [PARKED_FEATURES.md](PARKED_FEATURES.md).
- Related:
  - [Agent Tunnel](AGENT_TUNNEL.md)
  - [Published gateway](PUBLISHED_GATEWAY.md)
  - [Hops](HOPS.md)
  - [Internal CA](INTERNAL_CA.md)
  - Cloud-native access (design, next)
  - [Parked features](PARKED_FEATURES.md)

## Sites and connection profiles

A **Site** is a remote network: a name, a location and its subnets. A Site has one or more **connection
profiles**, the ways to reach it. With several profiles, the console uses the healthy one with the lowest
priority number (Agent Tunnel, then Direct). A profile can add **hops** (bastions, proxies) after it.

| Method | How it works | Status |
|---|---|---|
| **Direct** | The console uses routes your organization already has (VPN, MPLS, SD-WAN, ExpressRoute, Direct Connect, peering). Reachability probes and health per profile. | Active |
| **Bastion / SOCKS5 / HTTP CONNECT hops** | Console → profile → hop(s) → target; several hops, host-key pinning, SSH certificates | Active ([HOPS.md](HOPS.md)) |
| **Agent Tunnel** (session mode) | A connector in the Site dials out to the tunnel service (TLS, mutual TLS, WebSocket + yamux); per-session TCP to allow-listed targets; several connectors per Site (HA); HTTPS proxy support | Active ([AGENT_TUNNEL.md](AGENT_TUNNEL.md)) |
| Agent Tunnel via a **published gateway** | Only the tunnel service's connector port is published (firewall/DMZ, TCP passthrough); the console UI stays private | Active ([PUBLISHED_GATEWAY.md](PUBLISHED_GATEWAY.md)) |
| **Cloud-native access** | AWS SSM, Azure Bastion, GCP IAP (`cloud_native` profile type, reserved) | Coming soon (plans/cloud-native-access.md) |
| Site-to-Site IPsec, Point-to-Site, relay, subnet router, transit | Network infrastructure | Parked ([PARKED_FEATURES.md](PARKED_FEATURES.md)) |

### Choosing a method

The **New profile** pane has a guide that selects the type.

| Situation | Use |
|---|---|
| Your network already reaches the servers (VPN, MPLS, SD-WAN, cloud peering) | **Direct** |
| The servers sit behind a jump host | Direct, then add a **Bastion hop** |
| A corporate proxy is required | Direct, then add a **Proxy hop** (SOCKS5 or HTTP CONNECT) |
| No route from the console to that network, and the network team will not add one | **Agent Tunnel** connector |
| You need a site-to-site or user VPN | Set it up on your organization's network; then choose **Direct** |

### Sizing

- Direct and bastion/proxy: nothing to deploy beyond the tunnel service; sessions use your network.
- Agent Tunnel: **one or two connectors per Site**, not one per server. Two connectors share the load and take over from each other.
- Many hosts that need monitoring, discovery, ICMP or SNMP across whole subnets: route that network to the console on your side (VPN, peering) and use **Direct**.

### Containers

| Container | When | Privileges |
|---|---|---|
| `ce-atglance-tunnel` (tunnel service) | Any Remote Tunnel use: Direct and bastion sessions run here too (terminal, SFTP, forwards, SSH CA, probes, recordings, connector endpoint) | read-only, non-root, no Linux capabilities |
| Published connector port (`TUNNEL_PORT`, 8443) | Only with Agent Tunnel connectors (`docker-compose.tunnel-connectors.yml`) | — |
| Connector (`atglance/connector`) | At the remote Site, not next to the console | non-root, no capabilities |

No container needs `NET_ADMIN`, no TUN device, and no public UDP port. The tunnel service stays a separate
container (not part of the app): it holds long-lived WebSocket, SSH, SFTP and yamux streams that PHP
workers should not hold, and drains them on its own restart.

### Route resolution

`Sites::resolve($host)` answers "how does the console reach host X":
1. If X is a registered system assigned to a Site, that Site.
2. Otherwise, the Site with the longest matching subnet.
3. Within the Site, its best profile, then the profile's hops.

Systems whose address is in a Site's subnets are assigned to it automatically (every minute). The Site page
has a **Route test**. Console features use `TunnelDialer::forward()`, which opens an internal TCP forward
through that route.

Overlapping subnets across Sites are flagged on the Site page. Give the networks distinct addresses on your side (NAT ranges are part of the parked subnet router).

### Allow-lists

The allow-list is one entry per line, `CIDR ports` (for example `10.20.0.0/24 22,443`, `*` for every port).
`TunnelValidator` rejects malformed entries, control characters and overlapping networks. A session target
must be in the profile's allow-list. For an Agent Tunnel it must also be in the connector's own list.

## Sessions

| Kind | What the user gets |
|---|---|
| Terminal | xterm.js page (`/tunnel/term/<token>`). SSH to the target with the profile's saved target login: an SSH certificate from the console CA (recommended, nothing secret stored), an Ed25519 key, or a password (flagged weaker). |
| Web page (HTTP) | A redirect to `/tunnel/s/<token>/`. The tunnel service proxies HTTP to the target. |
| Files (SFTP) | A file page: browse, download, upload. The upload limit is set per profile (default 100 MB). Every transfer is audited with name, size and direction. |
| TCP forward | `tunnel:<port>` inside the console's Docker network, for API use or scripts. It closes when idle or at its maximum time. |
| Internal | The same as a TCP forward, opened by console features through `TunnelDialer` |

Target SSH host keys are pinned per profile and target: the first key seen is kept, and a different one is
refused. Reset a key under **Target logins** after a target is reinstalled.

### SSH certificates (console SSH CA)

The console keeps an Ed25519 SSH certificate authority (created on first use, the private key stored
encrypted; its public line is shown under **Remote Tunnel → Internal CA** and on a profile's **Target
logins** tab). A target login of kind **SSH certificate from the console CA** stores only the user name. For
each login the tunnel service signs a new key pair with a certificate that:
- is valid for 5 minutes;
- names only that user (principal);
- carries the session's ID and the console user in its key ID, so the target's `sshd` log shows who logged in.

Trust the CA once on each target (and on each bastion that uses certificate login):

```sh
echo 'ssh-ed25519 AAAA… atglance-console-ca' | sudo tee /etc/ssh/atglance_ca.pub
echo 'TrustedUserCAKeys /etc/ssh/atglance_ca.pub' | sudo tee -a /etc/ssh/sshd_config
sudo systemctl reload ssh
```

Bastion hops can use the same CA (authentication **SSH certificate**, see [HOPS.md](HOPS.md)). The CA is sent
to the tunnel service only while the instance is licensed.

### Terminal recording

With **Record terminal sessions** on a profile, the tunnel service records what the user sees in each
terminal (output and window size changes, asciicast v2). It does not record keystrokes, which can hold
passwords. When the session ends the recording is uploaded, gzip-compressed and HMAC-signed, to
`storage/app/tunnel-recordings/<session>.cast.gz` (up to 20 MB uncompressed, 10 MB compressed; a longer
session is cut and says so).

- The session's owner, and users who may end other users' sessions, play a recording from the sessions table
  (**Recording**) or download the `.cast` file (plays in `asciinema play`). Every view and download is in the
  activity log (`tunnel.recording_viewed`).
- `tunnel:recordings:prune` (daily, 03:20) deletes recordings older than `ATGLANCE_TUNNEL_RECORDING_DAYS`
  (default 90).
- Recordings are part of the storage volume and its backups.

`SessionBroker::open` checks, in order:
1. the licence;
2. the user's permission;
3. the allow-list;
4. the limits (5 per user, 20 per profile by default);
5. an online connector: the least busy of the profile's online connectors. The tunnel service fails over to the others when it cannot dial.

It then hands the tunnel service one session. The session has a single-use token (only its hash is
stored), the target, the limits and, for a terminal, the target login. The defaults are a 15-minute idle
timeout and an 8-hour maximum, both set per profile. `tunnel:sessions:reap` closes expired sessions every
minute. The `tunnel_sessions` table records the user, profile, target, port, start, end, reason and bytes.

## Permissions

| Who | Can |
|---|---|
| Administrator (`rbac_id` 100) | Everything: every Site and profile, the CA, the published gateway, the workspace policy, sharing organization profiles with workspaces |
| Workspace admin | The workspace's own Sites and profiles when **Manage Remote Tunnel connection profiles** is on and Remote Tunnel → Workspace policy allows that method for workspaces (one switch per profile type, and one for hops) |
| Workspace members | **Open Remote Tunnel sessions**, **End anyone's Remote Tunnel session** and **Reveal Remote Tunnel credentials** are separate workspace permissions, all off by default |

Pages: `/admin/tunnel` (organization) and `/admin/workspaces/{id}/tunnel` (workspace).

## App ↔ tunnel service

The app and the tunnel service share `ATGLANCE_TUNNEL_SECRET` (at least 32 characters). Each call in
either direction is signed:

```
X-Atglance-Timestamp: <unix seconds>
X-Atglance-Signature: hex(HMAC-SHA256(secret, "<ts>\n<METHOD>\n<path>\n<hex sha256(body)>"))
```

Clock skew may be at most 60 s. A request with a missing or wrong signature gets 404.

| Direction | Endpoint | Purpose |
|---|---|---|
| tunnel → app | `POST /api/internal/tunnel/bootstrap` | Server certificate chain, trust bundle, deny-list, connector allow-lists, licence state. Polled every 30 s. |
| tunnel → app | `POST /api/internal/tunnel/enroll`, `/renew` | Connector enrollment and certificate renewal (the tunnel service forwards the connector's CSR) |
| tunnel → app | `POST /api/internal/tunnel/connectors/status` | Online/offline every 15 s |
| tunnel → app | `POST /api/internal/tunnel/sessions/{uuid}/event` | Session closed, bytes |
| tunnel → app | `POST /api/internal/tunnel/sessions/{uuid}/hostkey`, `/file` | First SSH host key of a target (pinned); a file transfer (audit) |
| tunnel → app | `POST /api/internal/tunnel/hops/{id}/report` | A bastion host key to confirm, or a hop's last error |
| app → tunnel | `GET /internal/selfcheck`, `POST /internal/probe` | Published gateway check; reachability probes through a profile and its hops |
| app → tunnel | `POST /internal/sessions`, `/internal/sessions/{uuid}/close` | Open and close sessions |
| app → tunnel | `POST /internal/connectors/{uuid}/drop` | Revocation: drops the connection and refuses the certificate at once |
| app → tunnel | `POST /internal/drain` | Licence removed: close every session, connectors idle |
| ipsec → app (PARKED) | `POST /api/internal/tunnel/ipsec/sync` | Only with `ATGLANCE_TUNNEL_PARKED_IPSEC=true` ([PARKED_FEATURES.md](PARKED_FEATURES.md)) |

`/api/internal/*` uses the plugin manifest's `routes.internal` (docs/PLUGINS.md). The built-in Caddy proxy
answers 404 to `/api/internal` from outside. Caddy forwards `/tunnel/*` (terminal and port forwarding) to
`ATGLANCE_TUNNEL_UPSTREAM` (default `tunnel:8081`).

## Site → console access

A Site's connectors can open console endpoints **inside the Site**:
- the web console and API (local port 8000);
- the agent API used by the atglance CLI (Kong, local port 8002).

Servers and agents there use `<connector host>:<port>` (the `console.atglance.internal` DNS answer belongs to
the parked Site DNS). They never need the console's own address, and no inbound port opens on either
side. The endpoints are switched on per Site (Site page, **Console access**). The tunnel service forwards them
only to the fixed internal addresses (`TUNNEL_CONSOLE_UI` = `app:8080`, `TUNNEL_CONSOLE_AGENT` =
`gateway:8002`). Traffic and connection counts are shown on the Site page.

## Health, alerts and diagnostics

- **Health.** A profile's health comes from its connectors (Agent Tunnel) or probes (Direct). A Site's health comes from its profiles. The Remote Tunnel overview shows one card per Site.
- **Alerts** (Notifications, organization groups, and the workspace for workspace Sites):
  - connector offline;
  - Site down;
  - CA certificate expiring (30/7/1/0 days).
- **Diagnostics.** The Site page has a **Diagnostics** tab: licence → tunnel service → each profile (connectors, hops, probe targets) → whether the subnets are in an allow-list. On a connector host, `atglance-connector diagnose` checks DNS → TCP → proxy → TLS → mTLS.
- **Reachability.** On the profile page: TCP probes to chosen targets through the profile and its hops. For Direct profiles they run every 5 minutes and set the health.

## Licence removed

`tunnel:health` runs every minute even without the licence (`'always' => true`). When the licence no
longer includes `remote_tunnel`, the command:
- closes every open session;
- tells the tunnel service to drain (connectors stay enrolled but idle);
- records the downgrade once.

Profiles, connectors, certificates and target logins are kept. A banner explains the state. A licence that
includes the feature again restores sessions on the next tick, with no re-enrollment.

## Turning it off

**Site Settings → Plugins → Remote Tunnel → Disable** (super-admin, while licensed) does the same as a removed
licence, at once:
- sessions are closed;
- connectors go idle;
- its container is stopped: `ce-atglance-tunnel` (through the service controller, `ce-atglance-controller`). `tunnel:health` stops them again if an update starts them while the plugin is off;
- the sidebar item is hidden, and the Remote Tunnel pages send you back to the Plugins card.

Everything is kept: profiles, Sites, connectors, certificates and logins. **Enable** starts the container
again, and connectors resume within 30 seconds. Connectors on other hosts keep running and reconnect by themselves. Both are recorded in the activity log (`tunnel.plugin`).

## Testing

| What | Command |
|---|---|
| PHP | `composer/tests/Feature/RemoteTunnel*Test.php`, `composer/tests/Unit/TunnelValidatorTest.php` (normal PHPUnit run) |
| Go | `docker run --rm -v "$PWD/tunnel:/src" -w /src golang:1.23 go test ./...` |
| Go, parked code | `docker run --rm -v "$PWD/tunnel:/src" -w /src golang:1.23 go test -tags parked ./...` |
| End to end | `make tunnel-up`, `make tunnel-test` (Agent Tunnel), `PROFILE=hops`, `PROFILE=published-gateway`, `make tunnel-down`. Lab in `dev/tunnel/` (the `ipsec`, `relay` and `router` profiles are PARKED) |

The lab checks:
- enrollment and single-use tokens;
- HTTP forwarding;
- both allow-lists;
- the terminal;
- an admin ending a session;
- connector restart;
- a connector behind an HTTPS proxy;
- downgrade and relicensing;
- revocation.

## Known limitations

- The HTTP port forward strips cookies. It uses HTTPS for ports 443 and 8443, with certificate verification on.
- The console resolves host names for the allow-list check. Use IP addresses in profiles.
- Connector certificates last at least 24 hours, so the lab does not cover automatic renewal.
- A TCP forward listens on the tunnel service's Docker network address. Every container in that network can use it while the session lasts.
- NFS backups to an NFS server inside a Site are not routed through Remote Tunnel yet: the NFS plugin mounts from the console host, which has no route to the Sites.
- Terminal recordings hold the output only; input typed with echo off (passwords) is not in them, and neither is anything typed that the target does not echo.
