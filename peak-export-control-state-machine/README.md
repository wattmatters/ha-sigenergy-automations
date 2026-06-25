# SIGEN – Control Peak Export to Grid (State Machine)

A Home Assistant automation **shared as an example** — the state-machine-gated successor to the original [Peak Export Control](../archive/peak-export-control/) automation. It manages battery export during the evening peak feed-in bonus period, gated entirely on `input_select.battery_control_owner` rather than inferring ownership from time windows and Remote EMS state.

This is posted as a reference to adapt to your own system — not a plug-and-play solution. **Requires the [Battery Control Ownership State Machine](../battery-control-state-machine/) automation** to be running.

---

## What changed from the original

This is not a pure refactor — it includes a deliberate behaviour change found during the redesign review.

### The problem with `battery_export_enabled`

The original automation released to self-consumption whenever `input_boolean.battery_export_enabled` (set by the Battery Export Calculator Node-RED flow) went `off`. But that boolean goes false for **two unrelated reasons**:

1. Genuine battery protection — the flow's own minimum SOC floor for preserving the feed-in bonus, with the explicit stated rationale "protecting battery health, Zero Hero bonus at risk"
2. An unrelated **pre-window planning-phase projection** — the flow estimating, before the export window even opens, that projected exportable energy will be low based on overnight reserve, aircon, and solar forecast. This projection has nothing to do with battery protection and can flip rapidly with no debounce.

The original treated both cases identically — a full release of Remote EMS. This could needlessly abandon the configured Zero-Hero minimum export rate during otherwise-legitimate operation, just because an earlier planning-phase projection happened to be pessimistic.

### The fix

This version **replaces `battery_export_enabled` entirely** with a direct SoC check against the same floor the Node-RED flow itself uses for genuine battery protection (its Zero Hero minimum SOC):

- **RELEASE** fires only when SoC is at or below that floor
- **ACTIVE** runs at any SoC above that floor, regardless of what the planning-phase boolean says

Overnight-reserve protection now flows entirely through the Zero-Hero rate formula's own `soc_min` term (`input_number.battery_minimum_soc_end_of_export`, typically fed by an overnight load prediction automation) rather than through `battery_export_enabled`'s planning-phase branch — consolidating two parallel reserve-protection mechanisms into one.

### Structural simplifications enabled by the state machine

- The original's separate "switch ON" and "adjust while exporting" branches are collapsed into one state-guarded ACTIVE branch
- The time-window-based "outside period" cleanup branch is removed entirely — it's structurally unreachable once gated on `battery_control_owner`, which only equals `peak_export` during the configured window per the arbitrator's own logic. This makes a stale rate calculation past the end of the window unreachable *by construction*, rather than relying on a separate cleanup branch racing against the window boundary (a real bug class in the original design)
- The 6 PM debug persistent-notification branch is omitted — that was a diagnostic aid for the old time-window-guessing logic and has no equivalent need once ownership is explicit

---

## What it does

Gated on `battery_control_owner == 'peak_export'`:

- **RELEASE** — battery SoC at or below the protective floor: returns to Maximum Self-consumption mode and disables Remote EMS
- **ACTIVE** — battery SoC above the protective floor: enables Remote EMS, sets Command Discharging mode, and calculates an adaptive export rate every minute
- **Default** (SoC exactly at boundary, or sensor unavailable): releases safely rather than leaving stale state

Both ACTIVE and RELEASE force-reset the charging limit to the live rated charging power sensor on every cycle — peak export never uses charging, but must not strand that number for whichever automation takes over next (see the [state machine README](../battery-control-state-machine/)'s "baseline reset" principle).

---

## Rate calculation logic

Unchanged from the original — the goal is to export as much energy as possible during the peak window while ensuring the battery doesn't fall below a minimum SOC needed to cover overnight household consumption:

```
SOC headroom = current SOC% − minimum overnight reserve SOC%
Energy available = (SOC headroom / 100) × battery capacity kWh
SOC-limited rate = energy available / hours remaining
rate = max(Zero Hero minimum, suggested rate)
rate = min(rate, SOC-limited rate)
rate = min(rate, peak export maximum)
```

See the [original automation's README](../archive/peak-export-control/) for the full explanation of each term, and the [Zero Hero protection](../archive/peak-export-control/#zero-hero-protection) concept.

---

## Requirements

In addition to the [shared Sigenergy requirements](../README.md):

| Function | Example entity (sigenergy2mqtt) | What to look for |
|---|---|---|
| Battery control owner | `input_select.battery_control_owner` | Set by the state machine automation |
| Battery SOC | `sensor.sigen_0_inverter_1_battery_soc` | State of charge in % |
| Rated battery capacity | `sensor.sigen_0_inverter_1_rated_battery_capacity` | Used in the SOC-limited rate calculation |
| Rated charging power | `sensor.sigen_0_inverter_1_rated_charging_power` | Used in the baseline reset |
| Remote EMS toggle | `switch.sigen_0_plant_remote_ems` | Switch controlling remote EMS mode |
| Grid export limit | `number.sigen_0_plant_grid_max_export_limit` | Max grid export in kW |
| Operating mode | *(select entity)* | Select with Self-consumption / Command Discharging options |

### Helper entities

Same as the [original automation](../archive/peak-export-control/), plus the state machine's `input_select.battery_control_owner`.

---

## Installation

1. Ensure the [Battery Control Ownership State Machine](../battery-control-state-machine/) is installed and running.

2. Disable the original (non-state-machine) Peak Export Control automation to avoid both versions fighting over the same hardware.

3. Copy this automation's YAML to your `config/automations/` directory.

4. Find and replace all `← REPLACE` and `← ADJUST` markers — see the table in the YAML's header comment.

5. Reload automations or restart Home Assistant.

---

## Known limitations / notes

- The protective SoC floor (default 25%) should match whatever floor your suggested-rate calculator uses for genuine battery protection — if these two values drift apart, you could get inconsistent behaviour between the rate suggestion and the release decision.
- `battery_capacity_kwh` is read from the live rated-capacity sensor with a fallback, consistent with the rest of this collection.
- Tested on **Home Assistant OS 17.3 / Core 2026.5.2 / Supervisor 2026.05.0** with a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region, using the **sigenergy2mqtt** add-on integration.

---

## Contributing

Shared as a starting point. If you adapt it for a different tariff structure, inverter, or NEM region, feel free to open a PR or issue.

---

## License

MIT — do whatever you like, no warranty implied.
