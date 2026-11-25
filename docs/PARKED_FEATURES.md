# Parked Remote Tunnel features

**AtGlance provides secure, audited access to servers. The organization provides network connectivity.**

Organizations already connect their networks (site-to-site VPN, MPLS, SD-WAN, ExpressRoute/Direct
Connect, peering, the company's remote-access VPN), set up by their network team. Remote Tunnel uses that
connectivity instead of rebuilding it. The features below rebuilt network infrastructure and are
**parked**: switched off and unwired, with their code, database tables and tests kept, so each one can
come back. Marker in the code: `PARKED(remote-tunnel-simplify)`.

Active: Direct profiles, bastion/SOCKS5/HTTP CONNECT hops, sessions (terminal, SFTP, web page and TCP
forwards, console features), Agent Tunnel in session mode (connectors, HA, HTTPS proxy, published
gateway, site → console listeners), the SSH CA, host-key pinning, allow-lists, recordings, audit,
diagnostics and alerts for those. Cloud-native access (AWS SSM, Azure Bastion, GCP IAP) is planned next:
`cloud_native` is a reserved profile type, shown as "Coming soon" (docs/plans/cloud-native-access.md).

## The switches

One switch per feature, default `false` (parked). `true` turns the feature back on.

| # | Feature | Switch (console `.env`) | Go build | Images / overlay |
|---|---|---|---|---|
| P1 | Site-to-Site transit (Site ↔ Site through the console) | `ATGLANCE_TUNNEL_PARKED_TRANSIT` | `-tags parked` | router images (it rides on the subnet router) |
| P2 | Point-to-Site VPN (devices, groups, profiles, VPN pool) | `ATGLANCE_TUNNEL_PARKED_P2S` | `-tags parked` | `ce-atglance-ipsec`, `docker-compose.ipsec.yml` |
| P3 | Site-to-Site IPsec VPN (strongSwan gateway, config sheet, IPsec certificates/PSK, failover) | `ATGLANCE_TUNNEL_PARKED_IPSEC` | `-tags parked` | `ce-atglance-ipsec`, `docker-compose.ipsec.yml` |
| P4 | Subnet-router mode, NAT ranges, Site DNS (`*.atglance.internal`), policy matrix | `ATGLANCE_TUNNEL_PARKED_SUBNET_ROUTER` | `-tags parked` | `ce-atglance-tunnel:<v>-router`, `atglance/connector:<v>-router`, `docker-compose.router.yml` |
| P5 | Relay mode (relay service, relay protocol, relay dialing) | `ATGLANCE_TUNNEL_PARKED_RELAY` | `-tags parked` | `atglance/relay` |

P6, the site → console listeners on connectors (console UI/API and agent API inside a Site), is **kept**:
plain TCP listeners in the connector and streams in the tunnel service, no extra container or privilege.
Only its `console.atglance.internal` DNS answer belonged to the Site DNS (P4) and is parked; clients use
`<connector host>:<port>` or a name in the Site's own DNS.

Console: `config/tunnel.php` (`parked`), read through `App\Plugins\RemoteTunnel\Services\TunnelFeatures`.
Go: `tunnel/internal/features` and the `parked` build tag.

## What was disabled, and where the code lives

| Feature | Wiring that asks the switch | Code kept in |
|---|---|---|
| P1 transit | `InternalController::connectorStatus` (transit counts), Site page Transit tab, Site form, `Sites::validate` | `TunnelTransitFlow`, `tunnel/cmd/gateway/transit.go` |
| P2 Point-to-Site | routes `tunnel.vpn.*`, VPN pane and `vpn` view data, key clean-up in `tunnel:sessions:reap` | `VpnController`, `PointToSite`, `VpnProfiles`, `TunnelVpnDevice`, `TunnelVpnGroup`, `views/partials/vpn.blade.php`, `tunnel/cmd/ipsec/ra.go` |
| P3 IPsec | routes `*.ipsec.*`, `/api/internal/tunnel/ipsec/sync`, IPsec tab, profile type `ipsec_s2s` (validator, picker, workspace policy), IPsec health and alerts, IPsec alert events | `IpsecController`, `IpsecPolicy`, `IpsecSheet`, `TunnelIpsecState`, `views/partials/ipsec.blade.php`, `tunnel/cmd/ipsec`, `tunnel/cmd/gateway/ipsec.go` |
| P4 subnet router | routes `*.rules.*`, Policies pane, `mode = subnet_router` (validator, form), NAT range and Site DNS fields, bootstrap `router`/`dns`, NAT address translation in sessions | `PolicyController`, `PolicyMatrix`, `SiteNetwork`, `TunnelPolicy`, `tunnel/cmd/gateway/{router,dns,probe_l3}.go`, `tunnel/cmd/connector/{router,dns}.go`, `tunnel/internal/{tun,netcfg,packet,policy}` |
| P5 relay | routes `tunnel.relays.*`, Relays pane, the "through a relay" connector entry, relays in the bootstrap | `RelayController`, `TunnelRelay`, `tunnel/cmd/relay`, `tunnel/internal/relayproto`, `tunnel/cmd/gateway/relays.go`, `tunnel/cmd/connector/relay.go` |

Default builds use stubs for the parked Go code: `tunnel/cmd/gateway/parked_off.go` and
`tunnel/cmd/connector/parked_off.go` (`//go:build !parked`).

## Existing data

The migration `2026_11_19_000001_park_tunnel_network_features` disabled existing records of parked
features (nothing is deleted, the schema does not change) and remembered each change:
- IPsec profiles and subnet-router profiles: `status = disabled`; sessions on them are refused with
  "This connection method is no longer active. Use Direct with your organization's network, or a
  bastion/connector.";
- policy rules: `enabled = false`; relays: `status = disabled`;
- Point-to-Site groups and devices: `status = disabled`; Point-to-Site settings: `enabled = false`;
- Sites: `transit_enabled = false`.

The administrator sees a one-time notice on the Remote Tunnel page. `php artisan tunnel:parked` shows
the state; `--park` runs the disabling again.

## Re-enabling a feature

1. Console `.env`: set the switch, for example `ATGLANCE_TUNNEL_PARKED_IPSEC=true` (Point-to-Site also
   needs `_IPSEC=true`; transit needs `_SUBNET_ROUTER=true`). Recreate app, worker and scheduler
   (`docker compose up -d`).
2. Restore the records the upgrade disabled: `docker exec ce-atglance-app php artisan tunnel:parked --restore=ipsec`
   (one feature per call: `transit`, `p2s`, `ipsec`, `subnet_router`, `relay`).
3. Build the parked images (they are not published by default):
   `docker build -f tunnel/Dockerfile --target <ipsec|relay|gateway-router|connector-router> -t <image> tunnel`.
   The tunnel service itself must be built from the parked stage for IPsec, relay, router and transit
   (the `gateway-router` target, or add `-tags parked` to its `go build`).
4. Add the overlay again where the feature needs one: `docker-compose.ipsec.yml` (IPsec, Point-to-Site)
   or `docker-compose.router.yml` (subnet router, transit), and publish its ports (UDP 500/4500 for IPsec).
   Put them back in the updater's `COMPOSE_NAMES` (`scripts/atglance_update.sh`, `.ps1`) if updates should
   keep them.
5. Tests: the feature's PHP tests already run with its switch on (`EnablesParkedTunnelFeatures`); Go:
   `go test -tags parked ./...`. Lab profiles `ipsec`, `relay`, `router` in `dev/tunnel` (PARKED).
6. Docs: remove the PARKED banner from `IPSEC_VPN.md`, `POINT_TO_SITE.md`, `RELAY.md` or `MULTI_SITE.md`.
