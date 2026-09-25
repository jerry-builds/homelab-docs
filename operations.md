# Operations

## What auditing my own setup found

In September 2026 I wrote a full inspection of both hosts (hardware, storage, every listening port, every guest config) and read it back as if it were someone else's environment. It was not flattering. In order of risk:

1. **No scheduled backups.** vzdump had no jobs on either node, including for the identity, secrets and password-vault services.
2. **Docker's Engine API listening on the LAN without authentication** on two containers. Anyone on the LAN could have created a container with the host filesystem mounted.
3. **UPS connected but unmonitored** on both hosts. A long outage would have ended in a hard power cut.
4. **A VPN kill switch that didn't hold** (detail in [network.md](network.md)).
5. Smaller issues: overlapping dnsmasq ranges, two database guests with 4 GB root disks under 8 GB of RAM, six kernel series on a 74%-full root, and a handful of privileged containers with every device allowed.

The point of writing it down was to make the gaps impossible to ignore. It worked. Items 1 to 4, plus a tightened Tailscale policy, are fixed and each fix was tested. What changed and how it was proven is in **[hardening.md](hardening.md)**. The item 5 issues were fixed in turn.

## Backups

Daily application-level backups of Vaultwarden, Authentik, Infisical and cloudflared: database dumps and snapshots, config and secrets. Each file is checked, then encrypted with age to a key that is kept off the host, then written to a root-only folder on the data pool. 7 daily and 4 weekly runs are kept. A full restore was tested on a separate machine. See [hardening.md](hardening.md#1-backups) and [runbooks/restore-from-backup.md](runbooks/restore-from-backup.md).

## Power

NUT on both nodes. On low battery, every guest is stopped cleanly and then the host halts. See [runbooks/ups-power-loss.md](runbooks/ups-power-loss.md).

## Storage redundancy: the deliberate gap

The bulk pools have no parity. A drive failure loses that drive's files and nothing else, and the media on them is replaceable. The data that isn't replaceable (databases, vault, secrets, documents) lives on the guest NVMe and is backed up, encrypted, to the pool.

This was tested involuntarily. In July 2026 an 8 TB drive dropped off the bus, though SMART had passed on the last check. The pool kept serving from the remaining branches. The drive was commented out of fstab with a dated note, a replacement went in, and the pool was rebuilt with the new branch. The mountpoint names still lag the kernel device letters by one from the swap, which is why mounts are by UUID and not by device name.

## Monitoring

- smartmontools on both hosts.
- ProxMenux Monitor for a dashboard.
- ntfy for notifications.
- Proxmox notification mail goes through the host's Postfix relay.

There is no Prometheus stack.

## Regenerating this documentation

It's a point-in-time snapshot. Re-running the same inspection refreshes it: `pct list`, `pct config <id>`, `pct exec <id> -- ss -lntup`, `lsblk`, `pveversion -v`, `nft list ruleset`, and the OPNsense config export. The un-sanitized version lives in the data pool, not here.
