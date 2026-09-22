# Operations

## What auditing my own setup found

In September 2026 I wrote a full inspection of both hosts (hardware, storage, every listening port, every guest config) and read it back as if it were someone else's environment. It was not flattering. In order of risk:

1. **No scheduled backups.** vzdump had no jobs on either node. Backups of the identity, secrets and password-vault guests were the first thing fixed: a nightly vzdump job to the data pool for those four, then the firewall VM, then everything else on a weekly cadence.
2. **Docker's API listening on the LAN without TLS** on two containers, one of them privileged. Anyone on the LAN could have created a container with the host filesystem mounted. Bound to localhost.
3. **UPS connected but unmonitored** on both hosts. NUT installed and configured so the hosts shut down cleanly on battery.
4. **A VPN kill switch that didn't hold** (detail in [network.md](network.md)).
5. Overlapping dnsmasq ranges, two database guests with 4 GB root disks under 8 GB of RAM, six kernel series on a 74%-full root, and a handful of privileged containers with every device allowed. Each fixed in turn.

The point of writing it down was to make the gaps impossible to ignore. It worked.

## Storage redundancy: the deliberate gap

The bulk pools have no parity. A drive failure loses that drive's files and nothing else, and the media on them is replaceable. The data that isn't replaceable (databases, vault, secrets, documents) lives on the guest NVMe and is backed up to the pool.

This was tested involuntarily. In July 2026 an 8 TB drive dropped off the bus. SMART had passed on the last check. The pool kept serving from the remaining branches; the drive was commented out of fstab with a dated note, a replacement went in, and the pool was rebuilt with the new branch. The mountpoint names still lag the kernel device letters by one from the swap, which is why mounts are by UUID and not by device name.

## Monitoring

smartmontools on both hosts; ProxMenux Monitor for a dashboard; ntfy for notifications. No Prometheus stack. Proxmox notification mail goes through the host's Postfix relay.

## Regenerating this documentation

It's a point-in-time snapshot. Re-running the same inspection (`pct list`, `pct config <id>`, `pct exec <id> -- ss -lntup`, `lsblk`, `pveversion -v`, `nft list ruleset`, and the OPNsense config export) is how to refresh it. The un-sanitized version lives in the data pool, not here.
