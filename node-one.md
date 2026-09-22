# Node one: the main host

## Hardware

| Item | Detail |
|---|---|
| Board | Gigabyte B550 EAGLE WIFI6 |
| CPU | AMD Ryzen 9 5950X, 16 cores / 32 threads |
| Memory | 64 GB DDR4-3200, non-ECC |
| GPU | NVIDIA RTX 2080, 8 GB, shared into containers by device passthrough |
| NIC | Intel 82599ES 10 GbE SFP+ (plus onboard 2.5 GbE and Wi-Fi 6) |
| Boot | Samsung 960 EVO 250 GB NVMe |
| Guest disks | Samsung 990 PRO 2 TB NVMe |
| Bulk | Six SATA drives, 2 TB to 12 TB, HGST and Seagate |
| UPS | CyberPower CP1500 AVR over USB |

Proxmox VE 9.2 on Debian 13, single node, no cluster.

## Storage tiers

Three tiers, chosen for different failure and performance needs.

1. **Boot NVMe.** Standard Proxmox LVM layout: root, swap, and an unused `local-lvm` thin pool. `/var/log` runs from RAM via log2ram and syncs to disk, to cut write wear.
2. **Guest NVMe.** One LVM-thin pool holding every container's root disk. Thin provisioning means allocated space is oversubscribed on paper (about 1.7 TB across 28 guests on a 1.8 TB pool) but only written blocks consume space; actual use sits around 25%. A separate 500 GB LV on the same NVMe takes in-progress downloads so random writes stay off the spinning disks.
3. **Bulk HDD.** Six drives in two mergerfs pools. Both use `moveonenospc` and a `minfreespace` floor. The data pool uses `category.create=mfs` (most free space) so new files spread across disks; the media pool uses `epmfs` (existing path, most free space) so a library stays together on one disk. No RAID, no parity: losing a drive loses that drive's files and nothing else, which is a trade I made knowingly and document in [operations.md](operations.md).

The pools are not registered as Proxmox storage. Containers that need them get bind mounts (`mp0`, `mp1`) in their config with `backup=0`, so vzdump backs up the root disk and not 30 TB of media.

## Containers by function

28 LXC guests, most unprivileged. Grouped by what they do:

| Function | Guests |
|---|---|
| Databases | PostgreSQL x2, Neo4j x2, Redis, Qdrant |
| Identity and secrets | Authentik (SSO), Vaultwarden (password vault), Infisical (secrets manager) |
| Remote access | cloudflared (Cloudflare Tunnel), a connector container running both Twingate and Tailscale |
| Apps | Mattermost, Homarr dashboard, ntfy notifications, SearXNG, a Docker host for project workloads |
| AI | Ollama, Open WebUI, an API proxy (all stopped when not in use; they compete with everything else for RAM) |
| Media | Plex and Jellyfin, ebook and audiobook libraries |
| Home automation | openHAB (stopped; Home Assistant runs elsewhere) |

The four identity/secrets/ingress guests carry `protection: 1` so they can't be removed through the API by accident. Six guests have explicit `startup:` ordering so the databases come up before the things that need them.

Each guest has a fixed MAC in its config and gets its address from a DHCP reservation on the router, so addresses survive rebuilds without hard-coding them in Proxmox.

## GPU passthrough into LXC

The NVIDIA driver is installed on the host only. Containers that need the GPU get the `/dev/nvidia*` device nodes plus a read-only bind of the host's userspace libraries, so guest and host driver versions can never drift apart. Two styles coexist: the modern `devN:` syntax Proxmox 9 manages natively, and older raw `lxc.mount.entry` lines on guests built before that existed. Both work; new guests use `devN:`.

## Resource commitment

vCPU and RAM are oversubscribed on paper (79 vCPU and about 124 GB allocated to running guests on a 32-thread, 64 GB host). LXC only consumes what a guest actually uses, so this is normal; real usage at inspection was 14 GB with 37 GB of cache. It's a number to watch, not to fix.
