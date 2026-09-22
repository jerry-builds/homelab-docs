# Node two: the edge host

An older machine that exists to keep the network up when the main host is down for maintenance. It runs the firewall and the LAN's DNS.

## Hardware

| Item | Detail |
|---|---|
| Board | MSI Z97 GAMING 5 |
| CPU | Intel i7-4770K, 4 cores / 8 threads |
| Memory | 32 GB DDR3 (the board's maximum) |
| NICs | Two dual-port Intel X540 10 GbE cards, one quad-port Realtek 2.5 GbE card, onboard 1 GbE |
| Storage | 256 GB NVMe boot, two small SATA drives for guest disks |
| UPS | APC over USB |

Proxmox VE 9.1 on Debian 13. Mounts the main host's data pool over SMB 3 via systemd automount with `nofail`, so it boots cleanly if node one is off.

## Guests

| Guest | Role |
|---|---|
| OPNsense VM | Router and firewall for the whole LAN. 4 vCPU, virtio NICs, one on the LAN bridge and one on a VLAN-aware WAN bridge |
| AdGuard Home | LAN DNS resolver with ad and tracker blocking; static address; the one thing on the network everything else depends on |
| Nginx Proxy Manager | Reverse proxy with Let's Encrypt certificates for services published by hostname |
| Speedtest Tracker | Scheduled speed tests every six hours (stopped) |

Design detail on OPNsense, DNS and DHCP is in [network.md](network.md).
