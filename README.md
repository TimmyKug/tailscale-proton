# Tailscale exit node through Proton VPN

An isolated Docker Compose stack: phone → Tailscale → NAS → Proton WireGuard → internet.
The existing Tailscale installation on the NAS remains separate.

Android runs only Tailscale. The NAS runs the Proton connection, avoiding the
need for two Android VPN apps at once. This is an independent community project,
not an official Proton, Tailscale, or TrueNAS application.

## Requirements

- Linux NAS with Docker Compose and `/dev/net/tun`; Gluetun needs `NET_ADMIN`.
- A Proton WireGuard configuration downloaded from your Proton account.
- A Tailscale account with permission to approve an exit node.

## Setup

1. Copy `.env.example` to `.env`. Choose a unique `TS_HOSTNAME`.
2. Generate a WireGuard configuration in your
   [Proton account](https://protonvpn.com/support/wireguard-configurations).
   Create `secrets/` with mode 700. Save the config as `secrets/proton.conf`
   with mode 600. Do not commit it.
3. Run `docker compose config --quiet`, then `docker compose up -d`.
4. Run `docker compose logs tailscale` and open its Tailscale login URL.
5. In the [Tailscale admin console](https://login.tailscale.com/admin/machines),
   approve the device if it needs approval. Then open its **⋯ → Edit route
   settings**, enable **Use as exit node**, and save.
6. Select this exit node in the Android Tailscale app.

The config selects the specific Proton endpoint in your downloaded file. Replace
the config and recreate the stack to change servers. Endpoint must be an IP
address. Existing Proton configs may eventually need regeneration.

## TrueNAS custom app

Keep the project and credentials on persistent storage, outside the managed
`ix-apps` directories. Set absolute paths in `.env` for storage and the config.
Render the Compose configuration with:

```sh
docker compose config --format json > compose.rendered.json
```

Use the rendered configuration with TrueNAS **Apps → Discover → Install via
YAML**, or its `app.create` API (`custom_app: true`, `custom_compose_config`).
The rendered file contains paths and configuration, not the contents of your
WireGuard file. This is a custom app, not a published catalog application.

## Networking and verification

Tailscale uses userspace networking and shares Gluetun's network namespace.
Gluetun owns VPN routes and the firewall. Neither service publishes host ports,
and no LAN or tailnet bypass subnet is configured. Tailscale's resolver points
to Gluetun's local DNS service. IPv6 internet access must either work through
Proton or fail closed; test this on each deployment.

Userspace exit nodes support TCP, UDP, and proxied ICMP; throughput may be lower
than kernel routing. Peers may use Tailscale relays, particularly behind VPN NAT.

Before relying on it, verify from a separate Tailscale client:

- Public IPv4 belongs to Proton.
- IPv6 either belongs to Proton or cannot connect.
- DNS resolves through the exit node; review any Android Private DNS/browser DoH
  and tailnet DNS overrides separately.
- Blocking the Proton tunnel prevents new IPv4, IPv6, and DNS connections.
- Restoring the tunnel restores access.

Testing only an outbound request inside Gluetun is not a full exit-node test.

## Operations and recovery

Use the TrueNAS Apps controls when deployed as a custom app. With ordinary
Compose use `docker compose up -d`, `logs`, and `stop`. After updating/recreating
Gluetun, recreate both services together so they share the current namespace.

Back up `.env`, the Proton config and `data/tailscale/` securely. Tailscale state
contains the device identity: never share it between installations. Losing that
state requires logging in and approving a new node. Do not publish credentials,
state, or unreviewed logs.

To undo the installation, select no exit node on clients, stop/remove this custom
app without deleting its stored data, and remove its device in Tailscale. Existing
NAS services and host routing do not depend on this app.

## References

- [Tailscale Docker parameters](https://tailscale.com/docs/features/containers/docker/docker-params)
- [Tailscale userspace exit nodes](https://tailscale.com/kb/1177/kernel-vs-userspace-routers)
- [Gluetun WireGuard configuration](https://github.com/qdm12/gluetun-wiki/blob/main/setup/options/wireguard.md)
- [Proton WireGuard configuration](https://protonvpn.com/support/wireguard-configurations)

## Status

Tested on TrueNAS 25.10.7 with an Android client. The phone test was confirmed by
the operator. Container-level checks passed:
IPv4 and IPv6 public egress use the VPN; blocking its WireGuard endpoint prevents
new IPv4, IPv6, and DNS requests; removing the block restores access. Tailscale
authorization and exit-node route approval are complete. A full phone-level DNS
leak test and a disconnect test observed from the phone have not been performed.
Other NAS platforms have not yet been tested.

## License

MIT. Gluetun and Tailscale container images retain their respective licenses.
