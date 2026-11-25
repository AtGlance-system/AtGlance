# Bastion and proxy hops (Remote Tunnel)

A hop is an existing jump host or proxy between a connection profile and its targets. Add hops on the
profile page, **Hops** tab. They run in order:

```
console ─ profile (Agent Tunnel / Direct) ─▶ hop 1 ─▶ hop 2 ─▶ … ─▶ target
```

Examples:
- "Site A connector → bastion 10.0.0.5 → target 10.0.1.20:22";
- "Direct → SOCKS5 proxy → target".

| Type | How the tunnel service uses it | Authentication |
|---|---|---|
| **SSH bastion** (jump host) | An SSH connection to the bastion, then a `direct-tcpip` channel to the next hop or the target (OpenSSH `ProxyJump`) | An SSH certificate from the console SSH CA (recommended: 5 minutes, per session, nothing secret stored; trust the CA on the bastion with `TrustedUserCAKeys`, see [REMOTE_TUNNEL.md](REMOTE_TUNNEL.md#ssh-certificates-console-ssh-ca)), an SSH key, or user name and password (flagged as weaker). |
| **SOCKS5 proxy** | `CONNECT` per RFC 1928, by IP address or host name | None, or user name and password (RFC 1929). Optional TLS to the proxy. |
| **HTTP CONNECT proxy** | `CONNECT host:port HTTP/1.1` | None, or user name and password (Basic). Optional TLS to the proxy. |

## Rules

- **The profile must reach hop 1.**
  - Agent Tunnel: hop 1's address and port are in the connector's own allow-list.
  - Direct: the console host has a route to hop 1.
- **The target must still be in the profile's allow-list.** Hops never widen what a profile can reach.
- At most 4 hops per profile.
- SSH keepalives every 30 s on each bastion and target SSH connection (like OpenSSH `ServerAliveInterval 30`, `ServerAliveCountMax 3`): idle sessions survive firewalls and NAT that drop quiet connections, and a bastion that stops answering ends the session within about 90 s.
- Credentials are stored encrypted. They are sent only to the tunnel service, with each session, and never appear in logs or the activity log.
- Workspace admins can add hops only when the organization policy allows it (Remote Tunnel → Workspace policy → "Allow workspaces to add bastion and proxy hops").

## Bastion host keys

A bastion's SSH host key must be confirmed before the bastion is used (trust on first use, with an admin's
confirmation):
1. Open a session through the profile once. It fails with "host key SHA256:… not confirmed yet".
2. The Hops tab shows the fingerprint. Check it against the bastion (`ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`) and choose **Confirm**.
3. From then on, a different key is refused ("host key mismatch").

Targets reached at the end of the chain are pinned automatically: the first key seen is kept, and a change is
refused. Reset a target's key under **Target logins → Pinned SSH host keys** after it is reinstalled.

## Errors

Every failure names the hop, for example `hop 2 (bastion 10.0.0.5:22): SSH: unable to authenticate`. The
message appears:
- on the session;
- on the Hops tab (last error);
- in **Reachability** probes;
- in the Site's **Diagnostics**.
