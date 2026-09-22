# Homelab: two-node Proxmox environment

Documentation for the home lab I run in Oregon. Two Proxmox VE 9 hosts, 28 LXC containers, a virtualized OPNsense firewall, and the networking, DNS, remote access, identity and storage around them. Written from a live inspection of both hosts in September 2026, then trimmed for publication: addresses, hardware serials, credentials, and the internal domain are left out on purpose.

I keep this for the same reason I kept design docs at work. If I can't explain a design choice in writing, I don't understand it well enough to support it.

## Layout

| File | Covers |
|---|---|
| [node-one.md](node-one.md) | The main host: hardware, storage tiers, container inventory by function, GPU passthrough |
| [node-two.md](node-two.md) | The edge host: OPNsense VM, AdGuard Home, reverse proxy |
| [network.md](network.md) | Addressing, routing, firewall, DNS design, the isolated IoT subnet |
| [remote-access.md](remote-access.md) | Tailscale, Twingate and Cloudflare Tunnel, and why all three are there |
| [operations.md](operations.md) | Backups, monitoring, the drive failure, and what I changed after auditing my own setup |

## History

The lab ran on Unraid for two years: Docker containers on top of the Unraid array, with parity for the bulk storage. I moved to Proxmox in 2024 when I wanted proper Linux containers with their own IPs, a real virtualization layer for the firewall, and LVM-thin snapshots for guest disks. Unraid's storage model was the part I missed, which is why the bulk disks here are mergerfs pools rather than ZFS: I wanted the same "each disk stands alone" failure mode.

## Conventions

Most containers were created with the [community-scripts/ProxmoxVE](https://github.com/community-scripts/ProxmoxVE) helper scripts and carry a `community-script` tag. Where I changed something the script did, the per-container notes say so.

Nothing in this repo is inferred from intent. It was read from the running system with `pct`, `pveversion`, `lsblk`, `ss`, and the OPNsense config file, then rewritten by hand.
