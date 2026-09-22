# Network design

## Addressing

One flat LAN /24 for hosts, containers and trusted devices, plus a separate IoT /24 served by an access point on node one. No VLAN trunking to the switch; the IoT segment is isolated by being a different broadcast domain behind NAT.

Static addresses come from DHCP reservations on OPNsense, not from static configs on the devices. Around 70 reservations, named by role (`ct-` for containers, `iot-` for devices, `dev-` for workstations) so the lease table reads as an inventory.

## Routing and firewall (OPNsense)

- WAN via DHCP with private and bogon networks blocked inbound.
- Outbound NAT in hybrid mode.
- Hardware offloads (checksum, TSO, LRO) disabled, as recommended for virtio NICs.
- A WireGuard interface to a commercial VPN provider, configured with routes disabled so it's used only for policy routing. A small alias of devices is sent through that gateway by firewall rule, with a block rule beneath it as a kill switch. Getting the kill switch to actually hold required enabling "skip rules when gateway is down"; without it, OPNsense drops the gateway from the pass rule and the traffic leaks out the normal WAN. That one took a while to find.
- Inbound: a single port forward for a media server. Everything else reaches the LAN through the overlays in [remote-access.md](remote-access.md), so the firewall has no inbound rules for administration at all.

## DNS

Split DNS in two layers:

1. **AdGuard Home** is the resolver every device gets from DHCP. It filters, and it forwards the internal domain and reverse (PTR) lookups to Unbound.
2. **Unbound on OPNsense**, on a non-standard port, holds the host overrides for internal names and registers DHCP leases, so `hostname.internal` resolves for anything on the LAN. It forwards everything else upstream over DNS-over-HTTPS.

A firewall rule restricts LAN DNS to the two authorized resolvers, and a NAT redirect sends stray port-53 traffic to Unbound as a fallback, so a device with hard-coded public DNS still gets filtered answers.

## The IoT subnet

Node one's onboard Wi-Fi runs as an access point with hostapd on its own bridge. dnsmasq on that bridge hands out addresses and DNS; nftables masquerades the subnet out through the LAN bridge (deliberately not out through the radio, which is not an uplink). IoT devices can reach the internet and nothing on the LAN unless a rule allows it.

A lesson from documenting this: dnsmasq had picked up overlapping DHCP ranges from three config files added at different times, and its behavior with multiple `dhcp-range` lines on one interface is not obvious. They're now in one file.

## Host bridges

Node one: `vmbr0` (LAN, 10 GbE) and `vmbr1` (IoT AP). Node two: `vmbr0` (LAN, four 10 GbE ports bridged, STP off) and a VLAN-aware `vmbr1` for the firewall's WAN side. Only one guest per host has the Proxmox firewall enabled on its NIC (the internet-exposed media server), with a generated AppArmor profile and six capabilities dropped; that's the pattern I'm moving the other exposed guests toward.
