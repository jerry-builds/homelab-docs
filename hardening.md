# Hardening log

In September 2026 I audited my own lab as if it were someone else's (the findings are in [operations.md](operations.md)). This page covers what came next: every fix, what changed, and how I proved it worked. A fix doesn't count until it has been tested from the side an attacker or a failure would come from.

All seven items were completed and verified on 2026-09-25.

| # | Area | Before | After | Proof |
|---|---|---|---|---|
| 1 | [Backups](#1-backups) | No scheduled backups of the identity, secrets and vault services | Daily encrypted application-level backups, 7 daily + 4 weekly | Restore on a separate machine, every SHA-256 matched |
| 2 | [Docker APIs](#2-docker-apis) | Engine API on tcp/2375, no auth, on two containers | TCP listener removed | Connection refused from another host; nmap reports closed |
| 3 | [UPS](#3-ups-nut) | UPS on USB, nothing watching it | NUT on both nodes, guests stop cleanly, then the host halts | Both units reporting; shutdown path tested in dry-run mode |
| 4 | [Kill switch](#4-opnsense-vpn-kill-switch) | VPN-routed hosts leaked out the normal WAN when the tunnel dropped | Traffic blocked when the tunnel is down | Gateway forced down: no internet for routed host, LAN unaffected |
| 5 | [Subnet router](#5-tailscale-subnet-router-and-split-dns) | Basic route advertisement | Persistent forwarding, split DNS, LAN-hairpin fix | LAN service loaded by internal hostname from a phone on cellular |
| 6 | [Tailscale ACL](#6-tailscale-acl) | Default allow-all | Tag-based least-privilege policy with built-in tests | SSH to a LAN host blocked from outside, open from inside |
| 7 | [Exit node](#7-tailscale-exit-node) | Not in use | Advertised and auto-approved by policy | Public IP switches to the home WAN; no IPv6 leak |

---

## 1. Backups

**Why application-level, not container images.** The four services that would hurt most to lose are Vaultwarden, Authentik, Infisical and cloudflared. A whole-container image is large and restores the whole container or nothing. An application-level backup is small, captures the data in a consistent state, and can be restored onto a fresh container.

**What runs.** A systemd timer fires daily at 02:00 and:

- dumps the Postgres databases behind Authentik and Infisical;
- takes a consistent SQLite snapshot of Vaultwarden using its built-in `backup` command, not a file copy of a live database;
- collects each service's configuration and secrets;
- checks that every dump can be listed back and every archive is intact, **before** encrypting anything;
- encrypts every file with [age](https://age-encryption.org/) to a public key whose private key is **not on the host**;
- writes the result to a root-only folder on the data pool.

Plaintext exists only in RAM. Retention keeps 7 daily and 4 weekly runs, and a 100 GB soft cap stops a run rather than filling the pool.

**Proof.**

1. A manual run completed and produced seven encrypted files, about 32 MB in total.
2. The backup folder is readable only by root.
3. On a **separate machine**, every file decrypted and every SHA-256 checksum matched.

A backup that has never been restored is a hope, not a backup. The restore procedure is in [runbooks/restore-from-backup.md](runbooks/restore-from-backup.md).

## 2. Docker APIs

**Finding.** Two LXC containers exposed the Docker Engine API on tcp/2375 with no authentication. Anyone on the LAN could have started a container with the host filesystem mounted.

**Checking before changing.** Nothing should depend on that port, but "should" isn't evidence. I searched the configuration of every container on both nodes and watched live connections for ten minutes. No consumers, and the Authentik worker already used the local Unix socket.

**Change.** Removed the TCP listener from `daemon.json` on both containers (originals backed up) and restarted Docker.

**Proof.**

- No listener on 2375 inside either container.
- From another host, `nc` got connection refused and `nmap` reported the port closed.
- Docker still worked over its local socket. The Authentik containers were healthy again within about 25 seconds.

## 3. UPS (NUT)

**Setup.** NUT 2.8.1 on both nodes in netserver mode, with `upsd` bound to localhost only, the `usbhid-ups` driver, and `upsmon` as primary.

| Node | UPS | Low-battery trigger |
|---|---|---|
| Node one | CyberPower CP1500 AVR | UPS default |
| Node two | APC Back-UPS ES 600M1 | NUT declares low battery at 180 s of runtime or 50% charge |

The smaller APC unit gets an earlier trigger on purpose: its runtime is short, and the shutdown needs headroom.

**What happens on low battery.** `upsmon` calls a hook that stops every guest with `pvesh ... stopall`, allowing 150 s per guest before a forced stop, then halts the host. The sequence is in [runbooks/ups-power-loss.md](runbooks/ups-power-loss.md).

**Proof.**

- `upsc` showed both units online and reporting.
- `upsmon` confirmed logged in to `upsd` on each node.
- `upsdrvctl -t shutdown` (test mode) confirmed the command that would power off each UPS.
- A dry run of the guest-shutdown hook.

I deliberately did **not** trigger the real forced shutdown (`upsmon -c fsd`) on a running lab.

## 4. OPNsense VPN kill switch

**Finding.** A small group of hosts is policy-routed through a WireGuard tunnel to a commercial VPN provider, with a block rule beneath the policy rule as a kill switch. When the WireGuard gateway went down, the policy rule still matched and sent the traffic out the regular WAN. The kill switch never got a chance to act.

**Change.** Enabled "Skip rules when gateway is down" (Firewall, Settings, Advanced) on OPNsense 25.7, after taking a configuration backup.

**Proof.** With the WireGuard gateway marked down administratively:

- `curl` to two IP-echo services from a policy-routed test host timed out;
- ICMP to a public resolver showed 100% loss;
- LAN access and non-VPN hosts were unaffected.

With the gateway restored, the test host exited through the VPN provider again.

## 5. Tailscale subnet router and split DNS

**Change.** On the container running Tailscale:

- IPv4 and IPv6 forwarding made persistent;
- the LAN subnet advertised as a route and approved;
- split DNS sends the internal domain to AdGuard Home;
- MagicDNS confirmed on.

The Twingate connector on the same container kept running throughout.

**A problem I caused and fixed.** A Linux laptop that normally lives on the LAN accepted the new route, and its **local** traffic started going out through the tailnet instead of straight across the LAN. The fix was a persistent NetworkManager routing rule on the home Wi-Fi connection that sends LAN traffic direct. The route stays in place for when the laptop is away from home.

**Proof.** From an iPhone on cellular data, a LAN-only web service loaded by its internal hostname.

## 6. Tailscale ACL

**Change.** Replaced the default allow-all policy with a minimal one:

- the router container is tagged `tag:router`;
- my devices can reach each other, can use the exit node, and can reach the LAN subnet **only** on ports 53, 80, 443 and 8006 (DNS, web, Proxmox UI);
- everything else is denied by default;
- `autoApprovers` approves the subnet route and exit node for `tag:router`.

**Tests inside the policy.** The policy file has built-in tests. They must pass or the admin console rejects the policy:

- **allow:** DNS, HTTP and the Proxmox web UI;
- **deny:** SSH, the Authentik admin port and the Docker API port.

**Proof.** From the iPhone on cellular, the LAN web service still loaded by hostname, but SSH to a LAN host timed out. The same SSH port was confirmed open from inside the LAN, which shows the ACL was blocking it rather than something else.

## 7. Tailscale exit node

**Change.** The router container advertises an exit node, approved automatically by the policy's `autoApprovers`.

**Proof.** On the iPhone over cellular:

- with the exit node selected, an IP-echo service showed the home WAN address;
- with it deselected, the service showed the carrier's address.

The router has no IPv6 default route, so while the exit node was in use the phone fell back to IPv4 and did not leak over the carrier's IPv6.
