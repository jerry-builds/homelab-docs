# Runbook: restore a service from backup

**Use when:** Vaultwarden, Authentik, Infisical or cloudflared has lost data or its container has to be rebuilt.
**Backups:** daily at 02:00, age-encrypted, on the data pool in a root-only folder. 7 daily + 4 weekly are kept. How they are made is in [hardening.md](../hardening.md#1-backups).
**Last full restore test:** 2026-09-25, on a separate machine. Every file decrypted and every SHA-256 matched.

## Before you start

- [ ] Fetch the **age private key** from where it is kept offline. It is never stored on the lab hosts; that is the point.
- [ ] Decide what you're restoring to: the existing container, or a fresh one built from the same community script.
- [ ] Stop the service you're restoring, so nothing writes while you work.

## 1. Pick and copy the backup

Choose the newest run from **before** the problem started, not simply the newest file. Copy that run's encrypted files to a working folder on the machine doing the restore.

## 2. Decrypt and verify

```sh
age -d -i <private-key-file> <file>.age > <file>
sha256sum -c <checksum-file>
```

Every line must report `OK`. If any file fails, **stop** and try the previous run. Do not restore a partial set.

## 3. Restore, per service

| Service | What's in the backup | How to restore |
|---|---|---|
| Vaultwarden | SQLite snapshot from its own `backup` command, plus config | With the service stopped, put the snapshot in place of the database file, restore config, start |
| Authentik | Postgres dump, plus config and secrets | Load the dump into an empty database with the tool that matches its format (`pg_restore` or `psql`), restore config and secrets, start server and worker |
| Infisical | Postgres dump, plus config and secrets | As Authentik. The secrets must match the database, or stored values won't decrypt |
| cloudflared | Config and tunnel credentials | Restore both files, start the connector; the tunnel reconnects outbound |

## 4. Check it worked

- [ ] Service starts and stays up; logs are clean.
- [ ] **Vaultwarden:** log in, open a recently added item.
- [ ] **Authentik:** sign in to one downstream app through SSO.
- [ ] **Infisical:** read one known secret.
- [ ] **cloudflared:** the public hostname loads from outside the LAN.
- [ ] Delete the decrypted plaintext from the working folder.

## Notes

- Plaintext should never be written to the data pool. Decrypt only where you're restoring.
- If the private key is ever lost, existing backups cannot be read. Generate a new key pair, change the public key on the hosts, and take a fresh backup the same day.
