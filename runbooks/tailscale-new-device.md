# Runbook: add a device to the tailnet

**Use when:** setting up a new phone, laptop or workstation to reach the lab remotely.
**Design:** [remote-access.md](../remote-access.md) and [hardening.md](../hardening.md#5-tailscale-subnet-router-and-split-dns).

## What a new device gets

The ACL is written around my identity, not individual devices, so a new device signed in as me gets the same access with no policy edit:

- my other devices;
- the LAN subnet, **only** on ports 53, 80, 443 and 8006;
- the exit node.

Nothing else. SSH to LAN hosts is denied from the tailnet on purpose.

## Steps

1. Install Tailscale and sign in with my account.
2. **Linux only:** accept subnet routes. Mobile and macOS clients accept them by default.
   ```sh
   sudo tailscale set --accept-routes
   ```
3. **If the device also lives on the home LAN** (a laptop, say): add a persistent policy-routing rule on the home Wi-Fi connection so LAN traffic goes direct instead of through the tailnet. Without it, the device sends local traffic through the tailnet whenever it's at home. That's the problem I hit in [hardening item 5](../hardening.md#5-tailscale-subnet-router-and-split-dns).
4. Leave key expiry on unless the device is a server.

## Check it worked

From the device, **off** the home network (cellular or another Wi-Fi):

- [ ] `tailscale status` lists the router online.
- [ ] A LAN service loads by its **internal hostname**. That proves split DNS and the subnet route.
- [ ] SSH to a LAN host **times out**. That proves the ACL.
- [ ] Select the exit node: an IP-echo site shows the home WAN address. Deselect it: it shows the local network's address.

From **on** the home network (if applicable):

- [ ] LAN traffic goes direct. A traceroute to a LAN host shows one hop, not the tailnet.

## Gotcha

The home LAN uses the most common home /24. On someone else's network with the same range, the advertised route collides with their local one. The fixes are renumbering the home LAN or using Tailscale's 4via6 subnet routing. Both are noted in [remote-access.md](../remote-access.md); neither is done yet.
