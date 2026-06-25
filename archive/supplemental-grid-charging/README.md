# SIGEN – Control Supplemental Grid Charging

> **⚠️ Archived — superseded by the [state-machine version](../../supplemental-grid-charging-state-machine/).** This original version is kept for reference. See the [archive README](../README.md) for why it was superseded and when you might still prefer this simpler original.

A Home Assistant automation **shared as an example** for Sigenergy energy controller owners who want to automatically manage battery charging during a cheap or free electricity rate period.

This is posted as a reference to adapt to your own system — not as a plug-and-play solution. It was developed and tested on a specific system (see [Tested on](#known-limitations--notes)) and makes assumptions about time windows, helper entities, and operating modes that reflect that setup.

---

## What it does

Runs every minute during a configurable monitoring window and manages three scenarios:

- **Cheap period, charging required** — enables Remote EMS, switches to Command Charging mode, and sets the charge rate based on a suggested rate helper (with a configurable minimum floor)
- **Cheap period, charging not required** — resets charge rate to maximum, switches to Maximum Self-consumption mode, and disables Remote EMS to allow solar charging naturally
- **Outside cheap period** — ensures Remote EMS is off and the inverter is in Maximum Self-consumption mode, unless an AEMO critical override or grid bias offset automation is active

The decision of *whether* supplemental charging is required is made by a separate automation or manual toggle — this automation only manages *how* the charging is executed.

---

## Requirements

### Hardware — SIGEN inverter integration

See the [repository README](../../README.md) for details on the two supported integrations (sigenergy2mqtt and Sigenergy Local Modbus). The entities required here are:

| Function | Example entity (sigenergy2mqtt) | What to look for |
|---|---|---|
| Remote EMS toggle | `switch.sigen_0_plant_remote_ems` | Switch controlling remote EMS mode |
| Max charging limit | `number.sigen_0_plant_max_charging_limit` | Max charge rate in kW |
| Operating mode | *(select entity)* | Select with Self-consumption / Command Charging / Command Discharging options |
| Remote EMS control mode | `select.sigen_0_plant_remote_ems_control_mode` | Used to detect if another automation owns the EMS |
| Rated charging power | `sensor.sigen_0_inverter_1_rated_charging_power` | Maximum charge rate in kW — see note below |

### Capacity sensors — may need enabling

Some Sigenergy integration entities are disabled by default. If `sensor.sigen_0_inverter_1_rated_charging_power` does not appear in your system, go to **Developer Tools → Entities**, search for it, and enable it. The same applies to these related sensors:

| Sensor | Description |
|---|---|
| `sensor.sigen_0_inverter_1_rated_battery_capacity` | Total battery capacity in kWh |
| `sensor.sigen_0_inverter_1_rated_charging_power` | Maximum charge rate in kW (typically ~0.5 × capacity) |
| `sensor.sigen_0_inverter_1_rated_discharging_power` | Maximum discharge rate in kW |

If these sensors are unavailable, the automation falls back to `16.5 kW` — adjust this fallback value in the YAML to match your system.

### Helper Entities

Create these in **Settings → Devices & Services → Helpers**:

| Entity | Type | Notes |
|---|---|---|
| `input_boolean.battery_supplemental_charging_required` | Toggle | Set by another automation or manually — controls whether charging is active |
| `input_boolean.aemo_critical_override_active` | Toggle | Shared with the AEMO Critical Event automation |
| `input_number.battery_charge_suggested_rate` | Number | *(Optional)* Target charge rate from another automation — see below |

### Optional integration point

**`input_number.battery_charge_suggested_rate`** — if you have a separate automation that calculates an optimal charge rate (e.g. based on solar forecast, pricing, or how much charge is needed by a target time), write its output to this helper and this automation will use it as the target rate. A minimum floor of 2.0 kW is enforced during the cheap period regardless of the suggested rate.

If you don't have such an automation, set this helper to your preferred fixed charge rate and it will always charge at that rate.

---

## Installation

1. Enable the capacity sensors listed above if they are not already visible in your system.

2. Create all helper entities listed above.

3. Copy `sigen_control_supplemental_grid_charging.yaml` to your `config/automations/` directory, or paste into the Automation editor → Edit in YAML.

4. Find and replace all `← REPLACE` and `← ADJUST` markers:

| Placeholder | What to replace with |
|---|---|
| `YOUR_SIGEN_DEVICE_ID` | Device ID from your HA SIGEN device page |
| `YOUR_SIGEN_REMOTE_EMS_SWITCH_ENTITY_ID` | Entity ID of your Remote EMS switch |
| `YOUR_SIGEN_OPERATING_MODE_SELECT_ENTITY_ID` | Entity ID of your operating mode select |
| `sensor.sigen_0_plant_remote_ems_control_mode` | Your Remote EMS control mode select entity |
| Cheap period times (`10:59:50`–`14:00:00`) | Your tariff's cheap/free rate window |
| Monitoring window (`10:00:00`–`17:30:00`) | Adjust to suit your schedule |
| Minimum charge rate (`2.0`) | Your preferred minimum charge rate in kW during the cheap period |

5. Reload automations or restart Home Assistant.

---

## Known limitations / notes

- The automation deliberately does not act during the cleanup phase if Remote EMS is in `Command Discharging` mode — this prevents it from interfering with the Grid Bias Offset automation if that is also in use.
- The rated charging power sensor falls back to `16.5 kW` if unavailable. This is close to but slightly below the author's rated maximum of `16.8 kW` — a conservative cap. Adjust the fallback in the YAML to suit your system.
- The cheap period time window (11 AM–2 PM) reflects the author's tariff and **must** be adjusted to match your own.
- Tested on **Home Assistant OS 17.3 / Core 2026.5.2 / Supervisor 2026.05.0** with a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region, using the **sigenergy2mqtt** add-on integration.

---

## Contributing

Shared as a starting point. If you adapt it for a different tariff structure, inverter, or NEM region, feel free to open a PR or issue to share what you changed.

---

## License

MIT — do whatever you like, no warranty implied.
