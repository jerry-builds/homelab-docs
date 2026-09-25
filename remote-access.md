# Remote access

Three overlays, one job each. None of them needs an inbound firewall rule.

| Tool | Used for | Why this one |
|---|---|---|
| **Tailscale** | Reaching the LAN from my own devices: phone on cellular, laptop away from home | Device-to-device with NAT traversal; a subnet router means no client on every container |
| **Twingate** | Zero-trust access to specific services for specific identities | Resource-level policy without exposing a whole subnet; a useful comparison for how Tailscale ACLs differ |
| **Cloudflare Tunnel** | Public web ingress for a couple of services behind Cloudflare | Outbound-only connector, TLS and DDoS handled upstream, no port forward |

## Tailscale setup

Hardened and verified on 2026-09-25; the tests are in [hardening.md](hardening.md#5-tailscale-subnet-router-and-split-dns).

- **Subnet router.** One container on node one advertises the LAN /24, with IPv4 and IPv6 forwarding made persistent. It shares the container with the Twingate connector; both need `/dev/net/tun` passed through, and it's the smallest guest on the box.
- **DNS.** MagicDNS on. Split DNS sends the internal domain to AdGuard Home, so internal hostnames resolve from anywhere on the tailnet.
- **ACL.** Least privilege, replacing the default allow-all:
  - the router is `tag:router`;
  - my devices reach the LAN only on 53, 80, 443 and 8006;
  - `autoApprovers` handles the route and the exit node;
  - built-in policy tests must pass (allow DNS, web and Proxmox UI; deny SSH, the Authentik admin port and the Docker API).
- **Exit node.** Advertised for untrusted Wi-Fi and verified. With it on, traffic leaves through the home WAN, and there is no IPv6 leak because the router has no IPv6 default route.
- **Linux clients** need `tailscale set --accept-routes`; mobile clients take routes by default.
- **Gotcha: a laptop on the LAN.** Once it accepted the route, its local traffic went through the tailnet. Fixed with a persistent NetworkManager routing rule on the home Wi-Fi connection.
- **Known limitation.** The LAN uses the most common home /24, so on someone else's network with the same range the advertised route collides with the local one. The fixes are renumbering the home LAN or Tailscale's 4via6 subnet routing. Both are noted; I haven't renumbered yet.

Adding a device: [runbooks/tailscale-new-device.md](runbooks/tailscale-new-device.md).

## macOS notes

Tailscale on macOS ships in three variants:

- **Standalone,** from Tailscale's package server. Uses a System Extension that needs approval under Privacy & Security.
- **Mac App Store.** Uses a Network Extension inside the app sandbox.
- **Open-source, CLI-only `tailscaled`.**

Installing both GUI variants on one machine stops the extension from loading. The fix is to remove both, empty the trash, reboot, and reinstall one. On the GUI variants the CLI lives inside the app bundle, not on `$PATH`. `scutil --dns` shows whether MagicDNS is the resolver, and `dig @100.100.100.100 <name>` queries it directly.
