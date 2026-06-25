# SIGEN – Grid Bias Offset Export (Overnight & Idle)

> **⚠️ Archived — superseded by the [state-machine version](../../grid-bias-offset-export-state-machine/).** This original version is kept for reference. See the [archive README](../README.md) for why it was superseded and when you might still prefer this simpler original.

A Home Assistant automation **shared as an example** for Sigenergy energy controller owners who want to apply a small persistent export offset to eliminate metering bias imports that the inverter's internal control system cannot detect.

This is posted as a reference to adapt to your own system — not as a plug-and-play solution. It is the most complex automation in this collection and has the most prerequisites.

---

## What it does

**What is metering bias?** Metering bias occurs when the energy flow reported by the Sigenergy system's own metering does not match what the utility (electricity retailer) meter records. In practice, the inverter may believe it is at net zero or exporting, while the utility meter is recording a small import. This discrepancy arises because the Sigenergy system controls based on its own sensor readings and cannot see the utility meter directly.

This automation addresses that by applying a tiny forced export offset (e.g. 30 W) via Remote EMS Command Discharging mode, effectively shifting the inverter's operating point so the net grid flow as seen by the utility meter sits at or just below zero — eliminating the small phantom imports that would otherwise appear on your bill.

It operates in five scenarios:

- **Deactivate** — if we currently own Remote EMS (export limit is at bias-offset level) and any exit condition is true (battery full, managed period starting, EV charger starting)
- **Activate (battery full)** — battery is at ≥99.5% SOC but filtered grid power indicates sustained low or positive import; apply the offset
- **Update (battery full)** — already active in the above state; refresh the export limit
- **Activate (battery charging)** — battery SOC is below 99.5% (i.e. still charging); apply the offset to prevent metering bias during the charging period
- **Update (battery charging)** — already active in the above state; refresh the export limit

EV DC charging suppresses all activation branches — the automation will not take control while the DC charger is active, and a cooldown timer (shared with the [EV DC Charger Stop Cooldown Timer](../../ev-dc-charger-cooldown/) automation) prevents it from re-activating until the low-pass filter sensor has had time to settle after the EV load drops off.

---

## Why a filtered sensor?

Grid power readings from the inverter can fluctuate rapidly. Using the raw sensor directly would cause the automation to trigger and exit repeatedly on momentary spikes. The EWMA (Exponentially Weighted Moving Average) sensor smooths these fluctuations so the automation only responds to sustained conditions.

---

## Prerequisites

### 1. EWMA filtered grid power sensor

You need to create a **filter** helper sensor in Home Assistant that applies an exponential moving average to your grid active power reading.

**To create it:**
1. Go to **Settings → Devices & Services → Helpers → Create Helper**
2. Choose **Filter**
3. Configure as follows:

| Field | Value used in this system | Notes |
|---|---|---|
| Name | `EWMA - Grid Active Power` | Or similar |
| Input entity | `sensor.sigen_0_plant_grid_sensor_active_power` | ← REPLACE with your grid active power sensor |
| Filter | Exponential Moving Average | |
| Time constant | `10` | Higher = more smoothing, slower response — adjust to taste |
| Window size | `1` | |
| Precision | `2` | |

The resulting entity (e.g. `sensor.ewma_grid_active_power`) is what the automation uses. A time constant of `10` was used in this system — adjust to suit your system's noise characteristics. Higher values give more aggressive smoothing at the cost of slower response to genuine changes.

### 2. EV DC Charger Cooldown Timer

The `timer.ev_dc_charger_stop_cooldown` helper is shared with the [EV DC Charger Stop Cooldown Timer](../../ev-dc-charger-cooldown/) automation. If you are not using that automation, create the timer helper manually and start it from whatever automation stops your EV charger.

If you don't have a DC-coupled EV charger at all, remove the DC charger conditions from the YAML entirely.

### 3. Managed period flag

The automation references `input_boolean.managed_period_active` as an additional exit condition — a general-purpose flag indicating that another automation (charging, peak export, VPP schedule, etc.) has taken control of the inverter. Set this boolean from your other automations when they take ownership of Remote EMS.

> **Note on naming:** In the original system this boolean was named `input_boolean.red_energy_free_period` after an earlier energy retailer. It has been renamed here to a generic name. If you already have a boolean that serves this purpose under a different name, use that instead.

---

## Requirements

### Hardware — SIGEN inverter integration

See the [repository README](../../README.md) for details on the two supported integrations. The entities required here are:

| Function | Example entity (sigenergy2mqtt) | What to look for |
|---|---|---|
| Remote EMS toggle | `switch.sigen_0_plant_remote_ems` | Switch controlling remote EMS mode |
| Grid export limit | `number.sigen_0_plant_grid_max_export_limit` | Max grid export in kW |
| Operating mode | *(select entity)* | Select with Self-consumption / Command Charging / Command Discharging options |
| Remote EMS control mode | `select.sigen_0_plant_remote_ems_control_mode` | Used to confirm ownership of the EMS session |
| Battery SOC | `sensor.sigen_0_plant_battery_soc` | State of charge in % |
| Grid active power | `sensor.sigen_0_plant_grid_sensor_active_power` | Raw grid power in W (input to the EWMA filter) |
| DC charger running state | `sensor.sigen_0_dc_charger_1_running_state` | *(optional)* — remove if no DC charger |
| DC charger output power | `sensor.sigen_0_dc_charger_1_output_power` | *(optional)* — remove if no DC charger |

### Helper Entities

| Entity | Type | Notes |
|---|---|---|
| `sensor.ewma_grid_active_power` | Filter helper | EWMA over grid active power — see setup above |
| `input_number.grid_bias_offset_export_rate` | Number | Bias offset in kW (e.g. `0.03` for 30 W) |
| `input_boolean.grid_bias_offset_active` | Toggle | Flag indicating this automation is active |
| `input_boolean.aemo_critical_override_active` | Toggle | Shared with the AEMO Critical Event automation |
| `input_boolean.managed_period_active` | Toggle | Set by other automations when they own Remote EMS |
| `timer.ev_dc_charger_stop_cooldown` | Timer | Shared with the EV DC Charger Stop Cooldown Timer automation |

---

## Installation

1. Create all helper entities and the EWMA filter sensor as described above.

2. Copy `sigen_grid_bias_offset_export.yaml` to your `config/automations/` directory, or paste into the Automation editor → Edit in YAML.

3. Find and replace all `← REPLACE` and `← ADJUST` markers:

| Placeholder | What to replace with |
|---|---|
| `YOUR_SIGEN_DEVICE_ID` | Device ID from your HA SIGEN device page |
| `YOUR_SIGEN_REMOTE_EMS_SWITCH_ENTITY_ID` | Entity ID of your Remote EMS switch |
| `YOUR_SIGEN_OPERATING_MODE_SELECT_ENTITY_ID` | Entity ID of your operating mode select |
| `sensor.ewma_grid_active_power` | Your EWMA filter sensor entity ID |
| `sensor.sigen_0_plant_battery_soc` | Your battery SOC sensor |
| `sensor.sigen_0_plant_grid_max_export_limit` | Your grid export limit number entity |
| `select.sigen_0_plant_remote_ems_control_mode` | Your Remote EMS control mode select |
| `input_boolean.managed_period_active` | Your managed period flag boolean |
| `sensor.sigen_0_dc_charger_1_*` | Your DC charger sensors (or remove these conditions) |
| Time windows | Your cheap rate and peak export windows |
| `value: 5` (reset export limit) | Your maximum export limit in kW |
| `above: 99.4` / `below: 99.5` | Your battery full threshold |
| `< 0.6` (ownership check) | Slightly above your bias offset rate in kW |

4. Reload automations or restart Home Assistant.

---

## How the ownership check works

The automation identifies whether it currently owns the Remote EMS session by checking that:
- Remote EMS is `on`
- The mode is `Command Discharging`
- The export limit is below `0.6 kW` (i.e. at the tiny bias-offset level, not at a peak export level)

This prevents it from accidentally deactivating a peak export or AEMO override session. If your bias offset rate is higher than 0.5 kW, increase the `0.6` threshold accordingly.

---

## Known limitations / notes

- The `managed_period_active` boolean is an integration point — it needs to be set `on` by your other automations (charging, peak export, etc.) when they take ownership. Without this, the bias offset automation may attempt to activate during managed periods it doesn't know about via the hardcoded time windows.
- The EWMA time constant significantly affects behaviour. Too low and the filter won't smooth enough; too high and the automation will be slow to respond to genuine sustained import. Tune to your system.
- The 99.5% SOC threshold for "battery full" may need adjusting — some systems never quite reach 100% and will hover just below.
- Tested on **Home Assistant OS 17.3 / Core 2026.5.2 / Supervisor 2026.05.0** with a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region, using the **sigenergy2mqtt** add-on integration.

---

## Contributing

Shared as a starting point. If you adapt it for a different inverter or have refined the EWMA configuration, feel free to open a PR or issue to share what you changed.

---

## License

MIT — do whatever you like, no warranty implied.
