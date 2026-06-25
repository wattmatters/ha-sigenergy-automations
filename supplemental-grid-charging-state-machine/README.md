# SIGEN – Control Supplemental Grid Charging (State Machine)

A Home Assistant automation **shared as an example** — the state-machine-gated successor to the original [Supplemental Grid Charging](../archive/supplemental-grid-charging/) automation. It manages battery charging during a cheap or free electricity rate period, gated entirely on `input_select.battery_control_owner` rather than inferring ownership from time windows.

This is posted as a reference to adapt to your own system — not a plug-and-play solution. **Requires the [Battery Control Ownership State Machine](../battery-control-state-machine/) automation** to be running.

---

## What it does

Gated on `battery_control_owner == 'supplemental_charging'`:

- **ACTIVE** — grid charging required (sustained 2 minutes to avoid flapping on a noisy signal): enables Remote EMS, sets Command Charging mode, sets a charge rate based on the suggested rate with a minimum floor
- **SELF-CONSUMPTION** — grid charging not required (sustained 2 minutes): returns to Maximum Self-consumption, releases Remote EMS, and resets both the export and charging limits to maximum so genuine excess solar isn't capped

---

## What changed from the original

This is largely a structural refactor rather than a behaviour change — the charging decision logic itself is unchanged.

### Ownership gate replaces the cleanup branch

The original automation had a third branch: "outside free period — turn off Remote EMS", which existed purely to relinquish control reactively once the time window ended. The ownership gate (`battery_control_owner == 'supplemental_charging'`) replaces this entirely — outside the window, the gate simply doesn't pass, so the automation never acts in the first place. There is nothing to clean up reactively, because nothing was started reactively either.

### Baseline reset principle

Both branches above (ACTIVE and SELF-CONSUMPTION) unconditionally force-set three things on every cycle, following the cross-automation baseline-reset principle described in the [state machine README](../battery-control-state-machine/):

1. Its own switch/mode (state-guarded — only changed if not already correct)
2. **The grid export limit, reset to maximum every cycle** — this closes a real gap identified during the design review: if solar alone fills the battery before the window ends under the self-consumption branch, the export limit must not be left at a stale low value from a previous owner (e.g. peak export or an AEMO override). Genuine excess PV could otherwise be wrongly capped during a period where operational correctness still matters even though the feed-in tariff itself is near zero
3. **The charging limit, reset to the live rated-charging-power sensor** — not the original's hardcoded literal. This is the same number bias offset and peak export's parallel versions also force-set on entry, since none of the three exclusively owns it

### Hardcoded value replaced with live sensor

The original's charge-rate formula used a hardcoded `16.5 kW` as its upper bound. This version reads `sensor.sigen_0_inverter_1_rated_charging_power` live (with a fallback), matching the true rated maximum rather than a value that may drift out of date if the inverter firmware or configuration changes.

---

## Requirements

In addition to the [shared Sigenergy requirements](../README.md):

| Function | Example entity (sigenergy2mqtt) | What to look for |
|---|---|---|
| Battery control owner | `input_select.battery_control_owner` | Set by the state machine automation |
| Rated charging power | `sensor.sigen_0_inverter_1_rated_charging_power` | Used as the live upper bound, replacing the original's hardcoded value |
| Remote EMS toggle | `switch.sigen_0_plant_remote_ems` | Switch controlling remote EMS mode |
| Grid export limit | `number.sigen_0_plant_grid_max_export_limit` | Reset to maximum on every cycle |
| Max charging limit | `number.sigen_0_plant_max_charging_limit` | Set per the charge-rate formula |
| Operating mode | *(select entity)* | Select with Self-consumption / Command Charging options |

### Helper entities

Same as the [original automation](../archive/supplemental-grid-charging/), plus the state machine's `input_select.battery_control_owner`.

---

## Installation

1. Ensure the [Battery Control Ownership State Machine](../battery-control-state-machine/) is installed and running.

2. Disable the original (non-state-machine) Supplemental Grid Charging automation to avoid both versions fighting over the same hardware.

3. Copy this automation's YAML to your `config/automations/` directory.

4. Find and replace all `← REPLACE` markers — primarily entity IDs for your Sigenergy integration.

5. Reload automations or restart Home Assistant.

---

## Known limitations / notes

- The 2-minute sustained-state requirement on `battery_supplemental_charging_required` matches the original's dwell-lock fix for a previously-identified flapping bug — don't remove this debounce when adapting.
- The minimum charge rate during the free period (2.0 kW) is a fixed value — adjust to your preference.
- Tested on **Home Assistant OS 17.3 / Core 2026.5.2 / Supervisor 2026.05.0** with a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region, using the **sigenergy2mqtt** add-on integration.

---

## Contributing

Shared as a starting point. If you adapt it for a different tariff structure or inverter, feel free to open a PR or issue.

---

## License

MIT — do whatever you like, no warranty implied.
