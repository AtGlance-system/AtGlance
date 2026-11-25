# Published gateway (Remote Tunnel)

Use a published gateway when the console has only a private address or internal domain, but the console's site
can expose **one** port. Only the tunnel service is published, usually as `tunnel.company.com:443`. The console
UI and API stay private. Every Site's connectors dial out to this one address. See also
[AGENT_TUNNEL.md](AGENT_TUNNEL.md) and [REMOTE_TUNNEL.md](REMOTE_TUNNEL.md).

```
Site A connectors ─┐                     console site (private)
Site B connectors ─┼─ outbound 443 ─▶ [firewall / DMZ: TCP passthrough] ─▶ ce-atglance-tunnel:8443 ─▶ console
Site C connectors ─┘                     (no TLS termination)                  (mTLS)          (UI stays private)
```

The gateway accepts only connectors with a client certificate from the console's CA, plus `/enroll` with a
one-time token. Every other connection is refused before it reaches the console.

## 1. Settings on the console

In `.env`:

```bash
ATGLANCE_TUNNEL_PUBLIC_URL=wss://tunnel.company.com:443   # what connectors dial
TUNNEL_PUBLIC_NAMES=tunnel.company.com                     # names in the gateway certificate
TUNNEL_PORT=8443                                           # host port the firewall forwards to
```

Then run:

```bash
docker compose -f docker-compose.yml -f docker-compose.tunnel.yml -f docker-compose.tunnel-connectors.yml up -d
```

Check the gateway in **Remote Tunnel → Published gateway → Check from the console**, which tests DNS, TCP and
TLS from inside. Many firewalls don't allow hairpin NAT. In that case run the check from a Site instead:
`atglance-connector diagnose`.

## 2. Certificate

| Option | How | Connectors trust it by |
|---|---|---|
| **Internal CA** (default) | Nothing to do: the console's CA issues the gateway certificate for `TUNNEL_PUBLIC_NAMES` | The CA fingerprint in the enrollment token |
| **Public CA** (Let's Encrypt, your corporate CA trusted by the hosts) | Put `fullchain.pem` and `privkey.pem` in `./tunnel-certs/`, then set `TUNNEL_CERT_FILE=/etc/atglance-tunnel/certs/fullchain.pem` and `TUNNEL_KEY_FILE=/etc/atglance-tunnel/certs/privkey.pem` | The system's CA store. The console CA is still pinned: at enrollment the connector checks that the trust bundle it receives matches the token's fingerprint. |

The connector always presents its own certificate (mTLS), whichever option you choose.

## 3. Publish it: TCP passthrough only

The firewall or proxy must pass TLS through **unchanged**. A proxy that terminates TLS removes the connector's
client certificate, and connectors then fail with "no client certificate reached the gateway".

**Firewall port forward / NAT.** Forward `public-ip:443/tcp` to `console-host:8443/tcp`.

**nginx** (`stream` module, not `http`):

```nginx
stream {
    server {
        listen 443;
        proxy_pass console-host:8443;
    }
}
```

**HAProxy**:

```haproxy
frontend tunnel
    bind :443
    mode tcp
    default_backend console
backend console
    mode tcp
    server tunnel console-host:8443
```

**Traefik** (TCP router with TLS passthrough):

```yaml
tcp:
  routers:
    atglance-tunnel:
      rule: "HostSNI(`tunnel.company.com`)"
      tls: { passthrough: true }
      service: atglance-tunnel
  services:
    atglance-tunnel:
      loadBalancer: { servers: [{ address: "console-host:8443" }] }
```

**Cloud load balancers.** Use a TCP (layer 4) load balancer, not an HTTPS/application one:
- AWS: Network Load Balancer, TCP listener;
- Azure: Load Balancer (not Application Gateway);
- Google Cloud: TCP proxy or passthrough Network Load Balancer.

## 4. Connect Sites

Create an Agent Tunnel profile per Site and choose **Add connector**. The command it shows already uses
`ATGLANCE_TUNNEL_PUBLIC_URL`.

## Diagnostics

`atglance-connector diagnose` on a Site host:

| Message | Meaning and fix |
|---|---|
| `FAIL dns` | The name does not resolve at the Site. Fix DNS, or use the public IP address. |
| `FAIL tcp` | Outbound 443 is blocked, or the gateway isn't published. Open the firewall, or set `HTTPS_PROXY`. |
| `FAIL tls … not from the console CA` | Something terminates or inspects TLS on the way, or the address is wrong. Use TCP passthrough; exempt the address from TLS inspection. |
| `FAIL tls … does not name` | Add the host name to `TUNNEL_PUBLIC_NAMES`. |
| `FAIL mtls no client certificate` | A proxy terminates TLS. Switch it to TCP mode. |
| `FAIL mtls refused` | The connector was revoked or deleted. Enroll again. |
| `the proxy refused the CONNECT` | Allow the gateway's host and port on the proxy. |
