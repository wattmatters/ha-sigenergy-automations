# SIGEN – AEMO Critical Event Export Control

A Home Assistant automation **shared as an example** for Sigenergy energy controller owners in the Australian NEM who want to automatically maximise battery export during AEMO critical price events. It is posted here as a reference to adapt to your own system — not as a plug-and-play solution.

The automation was developed and tested on a specific system (see [Tested on](#known-limitations--notes)) and makes assumptions about time windows, helper entities, and operating modes that reflect that setup. You will almost certainly need to modify it to suit your own tariff schedule, inverter configuration, and broader Home Assistant automations.

---

## What it does

- **Detects AEMO critical price events** in real time using the AEMO NEM Web HACS integration
- **Pre-alerts** based on the 5-minute forecast price before an event officially starts
- **Automatically switches your SIGEN battery to maximum export mode** when a critical event is detected, subject to configurable time windows and a minimum battery reserve
- **Monitors every minute** during an active event: stops exporting at the battery reserve, and re-asserts Remote EMS, the operating mode and the export limit if any of them has drifted
- **Restores your normal operating mode** when the event ends, with context-aware handling for solar charging windows and peak export periods — *these time windows are specific to the author's system and will need adjusting*
- **Sends persistent notifications** at every state change so you always know what's happening

### Logic overview

```
Critical event detected?
  └─ Sufficient battery (>5 kWh)?
       ├─ Already exporting at max during peak window? → No action needed
       ├─ Solar/charging window (e.g. 11am–2pm) AND EMS on? → Override to export
       ├─ Remote EMS off? → Enable EMS, set export mode
       ├─ EMS on but export rate low? → Increase to maximum
       └─ Battery too low? → Notify only, don't export

Event ends?
  ├─ Always → Clear the critical event flag, log the price
  └─ Override took control?
       ├─ During solar window → Restore charging mode
       ├─ During peak window → Stay in export (no change needed)
       └─ Otherwise → Return to Maximum Self-consumption
       then → Clear the override flag (always the last step)
```

### Design rules

Four rules the automation follows. Keep them if you adapt it — each one exists because breaking it caused a real failure (see [Change history](#change-history)).

1. **Hardware first, bookkeeping last.** Logging the spot price to a helper is informational, so it runs after the inverter actions, is clamped to the helper's own min/max, and has `continue_on_error`. Nothing that only records information should be able to stop the export from starting.
2. **Clear `aemo_critical_override_active` last, on every exit path.** If you use the [Battery Control Ownership State Machine](../battery-control-state-machine/), it hands control back to the other automations the moment this flag turns off. Clearing it before the hardware restore has finished means two automations write to the inverter at once.
3. **Restore steps carry `continue_on_error`**, so one failed step cannot leave the override flag stuck on.
4. **After turning Remote EMS on, wait for the operating mode select to become available** before setting the mode, instead of using a fixed delay. The wait continues on timeout so the 1-minute monitor can re-assert the mode; aborting there would leave the override flag on with no mode set.

---

## Requirements

### Hardware — SIGEN inverter integration

This automation requires a Home Assistant integration for your SIGEN inverter. There are two common options; sensor naming differs between them, so you will need to identify the correct entity IDs in your own instance using **Developer Tools → States**.

#### Option A: sigenergy2mqtt (add-on)
[seud0nym/home-assistant-addons](https://github.com/seud0nym/home-assistant-addons/tree/main/sigenergy2mqtt) — a Mosquitto/MQTT-based add-on. This is the integration used when developing and testing this automation. Entity names follow the pattern used throughout this YAML (e.g. `sigen_0_plant_...`, `sigen_0_inverter_1_...`).

#### Option B: Sigenergy Local Modbus (HACS)
[TypQxQ/Sigenergy-Local-Modbus](https://github.com/TypQxQ/Sigenergy-Local-Modbus) — a HACS integration using local Modbus polling. Entity naming may differ from the examples in this YAML; use Developer Tools → States to find your equivalents.

Whichever integration you use, you need entities that expose:

| Function | Example entity (sigenergy2mqtt) | What to look for |
|---|---|---|
| Remote EMS toggle | `switch.sigen_0_plant_remote_ems` | A switch controlling remote EMS / grid export mode |
| Grid export limit | `number.sigen_0_plant_grid_max_export_limit` | A number entity for max grid export in kW |
| Max charge rate | `number.sigen_0_plant_max_charging_limit` | A number entity for max charge rate in kW |
| Operating mode | `select.sigen_0_plant_remote_ems_control_mode` | A select with options like Self-consumption, Command Charging, Command Discharging |
| Available discharge energy | `sensor.sigen_0_inverter_1_available_battery_discharge_energy` | A sensor for usable battery energy in kWh |

### HACS Integrations
- [**AEMO NEM Web**](https://github.com/pvandenh/aemo_nemweb) — install via HACS, providing:
  - `sensor.aemo_nemweb_<REGION>_realtime_price` ($/MWh)
  - `sensor.aemo_nemweb_<REGION>_5min_forecast` ($/MWh)
  - Replace `nsw1` in the YAML with your region: `vic1`, `sa1`, `qld1`, etc.

### Helper Entities
Create these in **Settings → Devices & Services → Helpers** before importing the automation:

| Entity | Type | Suggested default |
|---|---|---|
| `input_number.aemo_critical_threshold` | Number | 2.0 (= $2,000/MWh) |
| `input_number.aemo_pre_alert_threshold` | Number | 1.5 (= $1,500/MWh) |
| `input_number.aemo_nsw_spot_price` | Number | 0 (updated automatically) — **min −1000, max 30000**, see below |
| `input_boolean.aemo_critical_event` | Toggle | — |
| `input_boolean.aemo_critical_override_active` | Toggle | — |
| `input_boolean.aemo_pre_alert` | Toggle | — |
| `input_number.battery_charge_suggested_rate` *(optional)* | Number | your normal charge rate (kW) |

> **Set the spot price helper's range wider than the market can go.** `input_number.aemo_nsw_spot_price` stores the price in $/MWh. Give it a minimum at or below the market floor (−$1,000/MWh) and a maximum comfortably above the market price cap. The cap is indexed every July, so a maximum that was above it one year can be below it the next — 30000 leaves headroom. The automation clamps the logged value to whatever range you set, so a too-small maximum now only truncates the logged figure, but there is no reason to set it tight.

> **Create the two threshold helpers under exactly the names above.** The triggers fall back to hardcoded defaults ($3 and $2 per kWh) when a helper is missing, with no error or warning. If your thresholds seem to be ignored, check the entity IDs in the triggers against your helpers first.

### Optional: charge rate integration

`input_number.battery_charge_suggested_rate` is an **optional integration point** for users who have a separate automation that calculates an optimal battery charge rate (for example, based on a solar forecast, time-of-use tariff, or Amber Electric pricing). When a critical event ends during the solar/charging window, this automation reads that helper and restores whatever charge rate it recommends, rather than using a fixed value.

If you don't have such an automation, you have two options:
- **Create the helper anyway** and set it to your preferred fixed charge rate in kW — the automation will simply always restore to that value
- **Edit the YAML** and replace the template in the "Charging Restored" branch with a plain `value: X` (where X is your charge rate in kW), removing the dependency entirely

---

## Installation

1. **Create all helper entities** listed above.

2. **Copy `sigen_aemo_critical_event_export_control.yaml`** to your Home Assistant `config/automations/` directory, or paste its contents into the **Automation editor → Edit in YAML** view.

3. **Find and replace all `← REPLACE` placeholders** in the YAML:

   - **AEMO region**: change `nsw1` to your NEM region (`vic1`, `sa1`, `qld1`, `tas1`)
   - **SIGEN device_id**: open the SIGEN device page in HA, copy the device ID from the URL (`/config/devices/device/<id>`)
   - **SIGEN entity IDs**: use Developer Tools → States to find the exact IDs for your Remote EMS switch and operating mode select entity. Both are used twice: by their normal entity ID in conditions and the settle wait, and through the `YOUR_SIGEN_*` placeholders in the device actions
   - **Time windows**: adjust the `after`/`before` times in the `critical_event_start` and `critical_event_end` branches to match your tariff schedule and solar generation window
   - **Export/charge limits**: change the `value: 5` entries to match your inverter's actual maximum export capacity and your preferred settings

4. **Reload automations** (Developer Tools → YAML → Reload Automations), or restart Home Assistant.

---

## Configuration reference

| YAML placeholder | What to replace it with |
|---|---|
| `sensor.aemo_nemweb_nsw1_*` | Your region's AEMO sensor prefix |
| `YOUR_SIGEN_DEVICE_ID` | Device ID from your HA SIGEN device page |
| `YOUR_SIGEN_OPERATING_MODE_SELECT_ENTITY_ID` | Entity ID of the SIGEN operating mode select |
| `select.sigen_0_plant_remote_ems_control_mode` | The same operating mode select, by its normal entity ID (settle wait and monitor conditions) |
| `YOUR_SIGEN_REMOTE_EMS_SWITCH_ENTITY_ID` | Entity ID of `switch.sigen_0_plant_remote_ems` |
| `value: 5` (export limit) | Your inverter's maximum grid export in kW |
| Time windows (`17:59:50`–`20:00:00`) | Your peak export / VPP window |
| Time windows (`10:59:50`–`14:00:00`) | Your solar/cheap-rate charging window |
| `above: 5` / `below: 5` (battery reserve) | Your desired minimum battery reserve in kWh |

---

## How thresholds work

Prices from the AEMO NEM Web integration are reported in **$/kWh**, and the threshold helpers use the same $/kWh scale. The automation multiplies by 1,000 purely for display in notifications, converting to the **$/MWh** figures familiar from AEMO market reporting (e.g. a threshold of `2` $/kWh displays as $2,000/MWh in notifications).

Set your thresholds in $/kWh — for example `2.0` for a $2,000/MWh critical threshold. Adjust to suit your VPP agreement or personal threshold for when exporting makes financial sense.

The **pre-alert** triggers on the 5-minute forecast, giving you advance warning before the realtime price crosses the critical threshold. It doesn't change inverter behaviour — it just sets a flag and sends a notification so you can keep an eye on battery SoC.

---

## Notifications

The automation sends persistent notifications to your HA dashboard for every state change:

| Icon | Meaning |
|---|---|
| ✅ | Event ended or no action needed |
| ⚡ | Override active, exporting to grid |
| ⚠️ | Pre-alert or insufficient battery |
| 🛑 | Export stopped due to low battery |

---

## Known limitations / notes

- The automation is written for **SIGEN inverters** and uses device-action calls (by `device_id`) for the operating mode select. These cannot be replaced with simple entity service calls — you must supply your own `device_id`.
- Time windows are hardcoded. If your tariff windows change seasonally, update them accordingly or consider converting to `input_datetime` helpers.
- The automation assumes a **5 kW maximum export limit**. Adjust if your inverter or DNSP export limit differs.
- During an override the battery exports at the maximum rate down to the **5 kWh reserve** only. It does not apply any other floor you may use elsewhere, such as an overnight SoC target or a minimum export rate.
- If a critical event starts while the battery is already below the reserve, the run stops at the battery check: nothing is exported and the spot price is not logged for that event.
- The 1-minute monitor trigger means the automation runs every minute all day. `trace: stored_traces: 50` is set so the trace of a real event survives long enough to inspect; raise it if you need longer.
- The start, monitor and end paths have been checked against the configuration running on the author's system, but the October 2026 changes below have **not yet been exercised by a real critical event**.
- Tested on **Home Assistant OS 18.2 / Core 2026.7.4 / Supervisor 2026.07.5** with a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region, using the **sigenergy2mqtt** add-on integration.

---

## Change history

### October 2026 — override failed to start on a price above the helper's maximum

A 5-minute price of $23,200/MWh arrived while `input_number.aemo_nsw_spot_price` had a maximum of 21,500. Writing the price to that helper was the second step of the start sequence, ahead of every inverter action, so the rejected value aborted the run and nothing was exported. The critical event flag had already been set and then never cleared, because the end branch only ran when the override was active.

Changes made:

- **Price logging** moved after the inverter actions, clamped to the helper's range, with `continue_on_error` (design rule 1). Suggested helper maximum raised to 30000.
- **End branch** now always clears `aemo_critical_event` when the price drops back; the hardware restore still only runs if the override took control.
- **Override flag cleared last** on both exit paths (event end and battery reserve reached), with `continue_on_error` on the restore steps (design rules 2 and 3). Previously the flag was cleared first, which let the state machine's incoming automation run at the same time as this automation's restore.
- **Settle wait** replaces the fixed 2-second delay after turning Remote EMS on, and the mode is now set before the export limit (design rule 4).
- **Monitor** re-asserts Remote EMS, the operating mode and the export limit whenever any has drifted. Previously it did nothing unless Remote EMS was off.
- **`input_boolean.battery_export_enabled` removed** from the "already exporting" check. That helper is no longer required by this automation.
- **`trace: stored_traces: 50`** added.
- **YAML fixes in this published copy:** `← REPLACE` comments that sat inside template strings have been moved onto their own lines. Inside a template they are not YAML comments, and Home Assistant rejected the templates containing them. Top-level keys updated to the current `triggers` / `conditions` / `actions` form.

---

## Contributing

This is shared in the spirit of giving others a starting point — if you adapt it for a different inverter, NEM region, or tariff structure, feel free to open a PR or issue to share what you changed. That kind of context could save the next person a lot of time.

---

## License

MIT — do whatever you like, no warranty implied.
