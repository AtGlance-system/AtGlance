# Agent Tunnel

An Agent Tunnel connects a remote network without any inbound firewall rule. A small connector inside
that network keeps one outbound connection to the console's tunnel service. Sessions come back through
that connection. Overview, permissions and limits: [REMOTE_TUNNEL.md](REMOTE_TUNNEL.md).

Agent Tunnel runs in **session mode only**: each session is one TCP connection to an allow-listed target
(a "connector-based bastion"). Relay mode and the subnet router (whole subnets, NAT, Site DNS) are parked
([PARKED_FEATURES.md](PARKED_FEATURES.md)); for whole-network access, route that network to the console on
your organization's side and use a Direct profile.

```
remote network (no inbound)                    console host
[SSH :22] [web :443]                           ┌──────────────────────────────┐
     ▲ local connection                        │ ce-atglance-tunnel           │
[atglance-connector] ── wss :8443, mTLS ─────▶ │  :8443 connectors (mTLS)     │
   (HTTPS_PROXY ok)    one WebSocket, yamux    │  :8081 /internal, /tunnel/*  │ ◀── Caddy /tunnel/* ── browser
                                               └──────────────┬───────────────┘
                                                    HMAC      ▼
                                               app /api/internal/tunnel/*
```

## 1. Run the tunnel service

In the console's install folder, add these to `.env`:

```bash
ATGLANCE_TUNNEL_SECRET=<openssl rand -hex 32>   # same value for app and tunnel, 32+ characters
TUNNEL_PUBLIC_NAMES=console.example.com         # names in the tunnel service certificate (comma separated)
ATGLANCE_TUNNEL_PUBLIC_URL=wss://console.example.com:8443   # shown in the Add connector wizard
TUNNEL_PORT=8443                                # host port connectors dial
```

Then start the stack with the overlay:

```bash
docker compose -f docker-compose.yml -f docker-compose.tunnel.yml -f docker-compose.tunnel-connectors.yml up -d
```

The tunnel service (`ce-atglance-tunnel`) runs read-only, with no Linux capabilities and as a non-root
user. Only `TUNNEL_PORT` is published, by `docker-compose.tunnel-connectors.yml` (Direct and bastion-only installs leave that file out and publish nothing extra). Its internal port 8081 is reachable only from the Docker network. The
updater (`scripts/atglance_update.sh` / `.ps1`) keeps the overlay.

If the remote firewall allows only port 443, publish the tunnel service on 443 of a dedicated IP address or
host name. The console's own HTTPS proxy keeps 443 for the web console.

## 2. Profile and enrollment token

1. Go to **Remote Tunnel → New profile → Agent Tunnel**.
2. Set the allow-list, for example `10.20.0.0/24 22,443`.
3. On the profile page, choose **Add connector**. The console shows an enrollment token once:

```
atgct-<43 random characters>.<first 32 hex of SHA-256 of the console's root CA certificate>
```

The token works once and expires after one hour. Each profile can create at most 20 tokens per hour, and
enrollment is limited to 5 attempts per minute per source. The connector compares the fingerprint with the
CA that the tunnel service presents. A connector therefore never enrolls with the wrong console, even when
the TLS connection is intercepted.

## 3. Install the connector

**Docker**

```bash
docker run -d --name atglance-connector --restart unless-stopped \
  -v atglance-connector:/var/lib/atglance-connector \
  -e ATGLANCE_CONSOLE_URL=wss://console.example.com:8443 \
  -e ATGLANCE_ENROLL_TOKEN=atgct-… \
  -e ATGLANCE_ALLOW='10.20.0.0/24:22,443' \
  atglance/connector:1.0.0
```

**Linux (systemd)**: see the comments in `tunnel/packaging/atglance-connector.service`.

```bash
sudo install -m 0755 atglance-connector /usr/local/bin/
sudo install -m 0644 atglance-connector.service /etc/systemd/system/
sudo install -d -m 0700 /var/lib/atglance-connector /etc/atglance-connector
echo "10.20.0.0/24 22,443" | sudo tee /etc/atglance-connector/allow.conf
sudo atglance-connector enroll --console wss://console.example.com:8443 --token atgct-…
sudo systemctl enable --now atglance-connector
```

| Variable | Meaning |
|---|---|
| `ATGLANCE_CONSOLE_URL` | `wss://host:port` of the tunnel service |
| `ATGLANCE_ENROLL_TOKEN` | Used on the first start only (no `state.json` yet) |
| `ATGLANCE_ALLOW` | The connector's own allow-list: `CIDR:ports`, separated by `;` (`*` = every port). When this variable is not set, the connector reads `ATGLANCE_ALLOW_FILE` (default `/etc/atglance-connector/allow.conf`, `CIDR ports` per line, `#` comments) |
| `ATGLANCE_STATE_DIR` | Key, certificate and trust bundle (default `/var/lib/atglance-connector`, mode 0700) |
| `HTTPS_PROXY`, `NO_PROXY` | Outbound proxy (HTTP `CONNECT`) |

## How it works

1. **Enroll.** The connector makes an ECDSA P-256 key locally; the key never leaves the host. The connector sends a CSR (CN = a new connector UUID) and the token to `POST /enroll` on the tunnel service. The tunnel service forwards them to the app (`/api/internal/tunnel/enroll`). The app checks the token, signs the CSR with the [internal CA](INTERNAL_CA.md) (24 hours to 7 days, set per profile) and marks the token used.
2. **Connect.** `GET /connect` over TLS with the client certificate. The tunnel service checks the chain, the deny-list and that the connector is known. It then upgrades to WebSocket and runs yamux. The connector opens a control stream: allow-list updates, drain, renewal, errors.
3. **Dial.** For each session connection the tunnel service opens a yamux stream with a dial header. The connector checks the target against its own allow-list, dials it and answers with a dial result. A target outside the list is refused by the connector, even when the console allows it.
4. **Renew.** Before the certificate expires, the connector sends a new CSR on the control stream. The app signs it and revokes the old serial ("superseded"). The connector then reconnects with the new certificate.
5. **Reconnect.** The connector reconnects with exponential backoff and jitter. A restart needs no new token.

## Several connectors (high availability)

Run two connectors per Site, on different hosts. Use **Add connector** twice, one token each.
- New sessions go to the online connector with the fewest open sessions.
- If a connector cannot dial, the tunnel service tries the profile's other connectors.
- Sessions on a connector that dies break, and a new session goes to the other connector at once.
- The profile shows `degraded` while some of its connectors are offline. A notification is sent when a connector goes offline.

## Site → console access

On the Site page, under **Console access**, switch on the endpoints the Site may reach:
- the web console and API, on local port 8000;
- the agent API, on 8002.

Every connector of the Site then listens on those ports. Clients in the Site connect to the connector host,
and the connection goes through the tunnel to the console. To use a name, add an A record (for example
`console.example.internal → <connector host>`) in the Site's own DNS. (The connector's built-in DNS answer,
`ATGLANCE_DNS=1`, is part of the parked Site DNS and is ignored.)

The atglance CLI always uses ports 8000 and 8002 on the console host it is given, so keep the defaults
and give it the connector host (or your DNS name for it) as the console address.

| Variable | Default | Meaning |
|---|---|---|
| `ATGLANCE_CONSOLE_LISTEN` | all addresses | Address the local listeners bind to |
| `ATGLANCE_DNS`, `ATGLANCE_DNS_ADDR`, `ATGLANCE_DNS_ANSWER` | — | PARKED (Site DNS): ignored by the default connector |

Docker: publish the ports (`-p 8000:8000 -p 8002:8002`, and `-p 53:53/udp` for DNS), or use `--network host`.

## Diagnose

`atglance-connector diagnose` (Docker: `docker exec atglance-connector /usr/local/bin/atglance-connector
diagnose`) checks DNS → TCP (or the proxy) → TLS → that the console accepts this connector's certificate. For
each failure it prints a fix. Messages: [PUBLISHED_GATEWAY.md](PUBLISHED_GATEWAY.md#diagnostics).

## Revocation and removal

**Revoke** on the profile page:
- marks the connector revoked;
- adds its certificate serial to the deny-list;
- asks the tunnel service to drop it (`/internal/connectors/{uuid}/drop`).

The tunnel service closes the connection and refuses the serial at once, before its next 30-second
refresh. If the tunnel service is down, it gets the deny-list when it starts. To use the same host again,
enroll it with a new token.

## Troubleshooting

| Symptom | Check |
|---|---|
| Connector **Offline** | `docker logs atglance-connector` / `journalctl -u atglance-connector`. It must reach `ATGLANCE_CONSOLE_URL`. |
| `the console's CA does not match the token` | Wrong console URL, or TLS interception |
| `expected handshake response status code 101 but got 403` | Certificate revoked, or the connector was deleted on the console. Enroll it again. |
| `target not allowed by the connector` | Add the target to `ATGLANCE_ALLOW` / `allow.conf` |
| Tunnel features say "not configured" | `ATGLANCE_TUNNEL_SECRET` missing or shorter than 32 characters in the app or the tunnel service |
