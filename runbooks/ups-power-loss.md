# Runbook: power loss and UPS shutdown

**Use when:** power fails, when testing the UPS setup, or after replacing a UPS or its battery.
**Setup details:** [hardening.md](../hardening.md#3-ups-nut).

## What happens on its own

```
Mains fails
  -> UPS on battery; NUT reports status OB (on battery)
  -> Battery falls to the low-battery trigger (LB)
       node two (APC ES 600M1): 180 s runtime or 50% charge, set in NUT
       node one (CyberPower CP1500): UPS default
  -> upsmon runs the shutdown hook:
       pvesh ... stopall   (every guest, 150 s each before a forced stop)
  -> host halts
  -> UPS powers off its outlets
```

Node two holds the firewall and LAN DNS. Its smaller UPS gets the earlier trigger, so it shuts down while there is still headroom, not at the last second.

## Safe checks (nothing turns off)

Run on each node:

```sh
upsc <ups-name>                 # status OL = online, battery charge, runtime
upsdrvctl -t shutdown           # prints the power-off command; does not send it
```

Also confirm `upsmon` is logged in to `upsd`, and dry-run the shutdown hook.

**Do not** run `upsmon -c fsd` on a running lab. That is a real forced shutdown, not a test.

## When power comes back

- [ ] Hosts are up. If a host stays off, check the BIOS setting for power-on after AC loss.
- [ ] Node two first: OPNsense, then AdGuard Home. Nothing resolves until DNS is back.
- [ ] Node one guests start in their configured order: databases before the services that need them.
- [ ] Check the four identity and secrets services, and that Tailscale shows the router online.
- [ ] `upsc` shows OL and the battery is recharging.

## After replacing a UPS or battery

- [ ] `upsc` reports the new unit's model and runtime.
- [ ] Recheck the low-battery trigger against the new runtime. A bigger battery may allow a later trigger; a worn one needs an earlier one.
- [ ] Run the safe checks above.
