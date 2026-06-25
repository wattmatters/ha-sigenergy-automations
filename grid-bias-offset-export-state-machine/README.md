# SIGEN – Grid Bias Offset Export (State Machine)

A Home Assistant automation **shared as an example** — the state-machine-gated successor to the original [Grid Bias Offset Export](../archive/grid-bias-offset-export/) automation. It applies a small persistent export offset to eliminate metering bias imports, gated entirely on `input_select.battery_control_owner` rather than inferring ownership from Remote EMS switch/mode/export-limit readings.

This is posted as a reference to adapt to your own system — not a plug-and-play solution. **Requires the [Battery Control Ownership State Machine](../battery-control-state-machine/) automation** to be running. This is the most structurally complex automation in the collection, with a same-day post-cutover fix included below.

---

## What changed from the original

### Three-way split replaced with a single pass

The original automation had a three-way ACTIVATE/UPDATE/DEACTIVATE split, which existed only to infer prior state from indirect signals (was Remote EMS already on? was the export limit already at the bias-offset level?). This version replaces that with a **single unconditional per-cycle pass**: every trigger, decide RELEASE vs ACTIVE from scratch, and state-guard only the actual hardware actions (switch/mode), not the decision logic itself.

### Ownership gate replaces indirect exclusion checks

The original needed to separately check "is the DC charger active", "is the EV cooldown timer running", "is this outside the managed time windows" as standalone conditions, because it had no other way to know whether it should be operating. This version doesn't duplicate those checks as a separate condition block — they are exactly what `battery_control_owner == 'bias_offset'` already encodes via the arbitrator's own priority order (DC charging and cooldown are checked *inside* this automation's own RELEASE branches, since the arbitrator doesn't know about them, but the broader "is this even my time slot" question is answered once by the gate).

### entity_id targeting instead of device_id

The original used `device_id`-targeted actions for the Remote EMS switch and mode select. This version uses `entity_id` throughout — device_id targeting is fragile against device re-adds (re-pairing a Zigbee/Modbus device can issue a new device_id while keeping the same entity_id), so entity_id is the more robust choice.

### Solar-excess release now requires near-full SoC

A genuine bug fix folded into the cutover: the solar-excess RELEASE branch now additionally requires SoC above the near-full threshold before treating `solar_excess_available == on` as a release signal. Without this qualifier, the simple "SoC below near-full threshold" ACTIVE branch was being incorrectly blocked from ever activating at low SoC — because solar appearing to be "excess" relative to household load doesn't mean it's actually stranded if the battery still has charging headroom.

### Post-cutover fix: removed default-branch oscillation

**Fixed the day after the main cutover.** The original `default` branch (reached when neither ACTIVE nor RELEASE matched) released to self-consumption — turned the switch off. This created an oscillation: the EWMA-based ACTIVE branch's comparison (`filtered grid power > -bias_rate`) has no hysteresis. Once this automation is itself enforcing export at approximately the bias rate, the EWMA reading naturally settles right at that same threshold and flickers across it. During full-battery daylight conditions with no genuine solar-excess signal active, this branch was the *only* branch capable of matching — so the EWMA flicker directly flapped the switch on and off every few minutes via the old default branch.

The fix: the default branch is now a true no-op (baseline charge-limit reset only, no switch action). The three RELEASE branches are unaffected and still release normally on their own non-EWMA signals (solar excess, DC charging, EV cooldown) — only the bare "nothing else matched" fallback stopped touching hardware.

**If you adapt this automation and find yourself wanting to add a switch action to the default branch, stop and check whether what you're seeing is a genuine new state or just this same threshold-flicker pattern.**

---

## What it does

Gated on `battery_control_owner == 'bias_offset'`, runs a single decision pass each cycle:

| Branch | Condition | Action |
|---|---|---|
| RELEASE (solar excess) | Solar excess forecast on AND SoC near-full | Release to self-consumption |
| RELEASE (DC charging) | EV DC charger actively charging | Release — let EV see real excess solar |
| RELEASE (EV cooldown) | Cooldown timer active | Release — let sensors settle |
| ACTIVE (full battery) | SoC near-full, daylight hours, sustained low export per EWMA | Enable bias offset export |
| ACTIVE (simple) | SoC below near-full threshold, any time | Enable bias offset export |
| Default | None of the above matched | No-op (baseline reset only) |

See [What is metering bias?](../archive/grid-bias-offset-export/#what-it-does) in the original automation's README for an explanation of the underlying problem this solves.

---

## Requirements

In addition to the [shared Sigenergy requirements](../README.md) and the [original automation's prerequisites](../archive/grid-bias-offset-export/) (EWMA filter sensor, EV cooldown timer):

| Function | Example entity | Notes |
|---|---|---|
| Battery control owner | `input_select.battery_control_owner` | Set by the state machine automation |
| Solar excess available | `binary_sensor.solar_excess_available` | **New** — a template binary sensor; see below |

### New prerequisite: solar excess binary sensor

This version adds a forecast-based solar excess signal that the original didn't use. Create a template binary sensor:

```yaml
# Via Settings → Devices & Services → Helpers → Create Helper → Template → Binary sensor
# State template:
{{ states('YOUR_SOLAR_FORECAST_POWER_NOW_SENSOR') | float(0) >
   (states('YOUR_HOUSEHOLD_POWER_SENSOR') | float(0) + 200) }}
```

This compares a live solar forecast "power now" reading (e.g. from Solcast) against current household consumption plus a margin (200 W in this system — adjust to your own noise level). It is `on` when generation is forecast to comfortably exceed household load, signalling genuine excess solar rather than momentary noise.

---

## Installation

1. Ensure the [Battery Control Ownership State Machine](../battery-control-state-machine/) is installed and running.

2. Create the `binary_sensor.solar_excess_available` template sensor described above.

3. Disable the original (non-state-machine) Grid Bias Offset Export automation to avoid both versions fighting over the same hardware.

4. Copy this automation's YAML to your `config/automations/` directory.

5. Find and replace all `← REPLACE` markers.

6. Reload automations or restart Home Assistant.

---

## Known limitations / notes

- The default-branch fix described above is specific to systems using an EWMA-style filtered comparison with no hysteresis. If you redesign the ACTIVE branch's trigger condition to include hysteresis directly, you may be able to safely reintroduce a default release branch — but verify carefully against your own oscillation risk first.
- The near-full SoC threshold (99.4–99.5%) is shared across several conditions in this automation. Keep these consistent if you adjust them — a mismatch between the RELEASE and ACTIVE thresholds could create a small dead zone or overlap.
- Tested on **Home Assistant OS 17.3 / Core 2026.5.2 / Supervisor 2026.05.0** with a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region, using the **sigenergy2mqtt** add-on integration.

---

## Contributing

Shared as a starting point. If you adapt it for a different inverter or have a different approach to the oscillation problem, feel free to open a PR or issue.

---

## License

MIT — do whatever you like, no warranty implied.
