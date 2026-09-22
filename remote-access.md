# Remote access

Three overlays, one purpose each. None of them needs an inbound firewall rule.

| Tool | Used for | Why this one |
|---|---|---|
| **Tailscale** | Reaching the LAN from my own devices: phone on cellular, laptop away from home | Device-to-device with NAT traversal, and a subnet router means I don't install a client on every container |
| **Twingate** | Zero-trust access to specific services for specific identities | Resource-level policy without exposing a whole subnet; good comparison point for how Tailscale ACLs differ |
| **Cloudflare Tunnel** | Public web ingress for a couple of services behind Cloudflare | Outbound-only connector, TLS and DDoS handled upstream, no port forward |

## Tailscale setup

- One container on node one is the subnet router, advertising the LAN /24 (approved in the admin console). It shares a container with the Twingate connector; both need `/dev/net/tun` passed through, and it's the smallest guest on the box.
- Linux clients need `tailscale set --accept-routes`; they don't take subnet routes by default. Mobile clients do.
- MagicDNS on. ACL policy tags the router and limits which devices can use it.
- An exit node is advertised for use on untrusted Wi-Fi.
- Known gotcha reproduced on purpose: the LAN uses the most common home /24, so on someone else's network with the same range the advertised route collides with the local one. The fix is either renumbering the home LAN or using Tailscale's 4via6 subnet routing; I've noted both and haven't renumbered yet.

## macOS notes

Tailscale on macOS ships in three variants (Standalone from Tailscale's package server, Mac App Store, and an open-source CLI-only `tailscaled`). The Standalone build uses a System Extension and needs approval under Privacy & Security; the App Store build uses a Network Extension inside the app sandbox. Installing both on one machine stops the extension from loading, and the fix is remove, empty trash, reboot, reinstall one. The CLI on the GUI variants lives inside the app bundle, not on `$PATH`. `scutil --dns` shows whether MagicDNS is the resolver; `dig @100.100.100.100 <name>` queries it directly.
