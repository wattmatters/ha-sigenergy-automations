# Battery Charging Calculator

A Node-RED flow **shared as an example** that calculates an optimal battery charging strategy during a cheap or free electricity rate period, using Solcast solar forecasts, current battery state, and household consumption.

This is posted as a reference to adapt to your own system — not a plug-and-play solution. Time windows, consumption estimates, and capacity values are all specific to the author's system and must be adjusted.

---

## What it does

Polls eleven Home Assistant sensors every 60 seconds, joins them into a single object, and passes them to a JavaScript function that calculates a charging strategy. The result is written back to four HA helper entities consumed by the [Supplemental Grid Charging](../../supplemental-grid-charging-state-machine/) automation.

### Charging modes

| Mode | Description |
|---|---|
| `BATTERY_FULL` | Battery at capacity — no action needed |
| `GRID_CHARGING_PV_BUSY` | PV fully consumed by loads or EV charger — charge from free grid |
| `GRID_SUPPLEMENTAL` | Solar insufficient for deficit — supplement with grid |
| `OPPORTUNISTIC` | Small surplus — top up opportunistically from grid |
| `SOLAR_ONLY` | Large surplus — solar will fill battery naturally |
| `EV_CHARGER_KEEPALIVE` | Battery full but EV active — keep Remote EMS on for free grid EV charging |
| `EV_CHARGER_GRID_SUPPLEMENTAL` | Battery needs charging and EV active — charge both from free grid |
| `PLANNED_*` | Planning phase (before free period) — forecasting what will happen |
| `AFTER_FREE_PERIOD` | Free period has ended |

### EV charger latch

When an EV is detected (by running state or measurable power), a latch timer holds the `evChargerActive` flag true for a configurable number of minutes after the EV stops. This prevents the system from dropping out of Remote EMS during brief pauses (e.g. cell balancing) mid-charge.

### Dwell-time lock

A 5-minute lock on `gridChargingRequired` flips prevents rapid on/off toggling of `input_boolean.battery_supplemental_charging_required` when sensor values sit near a threshold boundary (e.g. `availablePVForBattery` oscillating across the 0.5 kW line). A flip is only accepted once `GRID_CHARGING_DWELL_MINUTES` has elapsed since the last accepted flip. The reason string is prefixed with `[Hold Xmin - dwell lock]` when a flip is suppressed, making it visible in the HA dashboard.

### Household power staleness guard

If `household_power` hasn't updated within `HOUSEHOLD_POWER_MAX_AGE_MINUTES` (default 5 minutes), the flow treats it as stale and sets `availablePVForBattery` conservatively to zero. This guards against a scenario where a firmware push or Modbus link drop causes the sensor to latch on its last value rather than going unavailable, which would silently feed a stale reading into the charge-rate calculation. The reason string is prefixed with a warning when staleness is detected.

A dedicated **Household Power Timestamp** Change node extracts the `last_updated` timestamp from the household power entity and passes it to the join node as a separate topic (`household_power_last_updated`), enabling this staleness check.

---

## Configuration

Open the **Calculate Charging Strategy** function node and edit the constants at the top:

| Constant | Default | Description |
|---|---|---|
| `DAILY_HOUSEHOLD_CONSUMPTION` | `30` | ← REPLACE: your estimated daily consumption in kWh |
| `FREE_PERIOD_START_HOUR` | `11` | ← REPLACE: cheap/free period start hour (24h) |
| `FREE_PERIOD_END_HOUR` | `14` | ← REPLACE: cheap/free period end hour (24h) |
| `MAX_CHARGING_RATE_KW` | `16.8` | ← REPLACE: your inverter's maximum charge rate in kW |
| `FREE_PERIOD_MIN_CHARGE_RATE` | `2.0` | Minimum charge rate during free period in kW |
| `CHARGE_RATE_DEADBAND_KW` | `0.5` | Prevents rate changes smaller than this value |
| `PV_SMOOTHING_FACTOR` | `0.3` | EMA smoothing on PV reading (0=none, 1=full) |
| `EV_LATCH_MINUTES` | `20` | ← REPLACE: minutes to hold EV active flag after charger stops |
| `GRID_CHARGING_DWELL_MINUTES` | `5` | Minimum minutes between accepted gridChargingRequired flips — increase if flapping still observed |
| `HOUSEHOLD_POWER_MAX_AGE_MINUTES` | `5` | Maximum age of household power reading before it is treated as stale |

---

## Input sensors

Replace all `YOUR_` entity IDs in the poll-state nodes with your own:

| Topic | Node name | Entity to use |
|---|---|---|
| `battery_soc` | Battery SoC % | Your battery SOC sensor (e.g. `sensor.sigen_0_inverter_1_battery_soc`) |
| `solar_remaining` | PV Forecast - Remaining Today | `sensor.solcast_pv_forecast_forecast_remaining_today` |
| `solar_this_hour` | PV Forecast - This Hour | `sensor.solcast_pv_forecast_forecast_this_hour` |
| `solar_next_hour` | PV Forecast - Next Hour | `sensor.solcast_pv_forecast_forecast_next_hour` |
| `household_power` | Current Household Consumption | Your total load power sensor (W) — see note below |
| `household_power_last_updated` | Household Power Timestamp | Extracted automatically from the household power entity's `last_updated` attribute via a Change node — no separate sensor needed |
| `sunset_hour` | Sunset time | `sensor.sun_next_setting` |
| `pv_generation` | PV Output Smoothed | Your smoothed PV power sensor (W) — see [node-red README](../README.md) |
| `dc_charger_power` | DC EV Charger Output Power | Your DC charger output power sensor (W) — remove if no DC charger |
| `dc_charger_state` | DC EV Charger Running State | Your DC charger running state sensor — remove if no DC charger |

**Note on household power sensor:** the sigenergy2mqtt integration exposes two relevant sensors — `sensor.sigen_0_consumed_power` and `sensor.sigen_0_total_load_power`. This flow uses `total_load_power` (← REPLACE with whichever your integration exposes). The staleness guard relies on the entity's `last_updated` timestamp being passed through the **Household Power Timestamp** Change node — ensure this node is wired correctly (the household power poll-state node feeds both the join node and the Change node, which in turn also feeds the join node with the timestamp as a separate topic).

**Note on join node:** The join node is configured to wait for **10 inputs** (9 sensors + household power timestamp). If you remove the DC charger nodes, reduce this count accordingly — double-click the join node and adjust the **Count** field.

---

## Output helpers

The flow writes to these HA helpers (create them before deploying):

| Entity | Type | Consumer |
|---|---|---|
| `input_boolean.battery_supplemental_charging_required` | Toggle | Supplemental Grid Charging automation |
| `input_number.battery_charge_suggested_rate` | Number | Supplemental Grid Charging automation |
| `input_text.battery_charge_strategy` | Text | Informational / dashboard display |
| `input_text.battery_charge_urgency` | Text | Informational / dashboard display |

---

## Known limitations / notes

- The `DAILY_HOUSEHOLD_CONSUMPTION` constant is a fixed estimate. If your household consumption varies significantly by season, consider making this an `input_number` helper so it can be adjusted without editing the flow.
- The `MAX_CHARGING_RATE_KW` constant can alternatively be read from `sensor.sigen_0_inverter_1_rated_charging_power` if available — add a poll-state node for it and reference `msg.payload.max_charge_rate` in the function.
- If DC charger nodes are removed, update the join node count from 10 to 8.
- The dwell-time lock (5 minutes default) prevents the flow from responding faster than that to genuine threshold crossings. If your free period is short or conditions change rapidly, consider reducing `GRID_CHARGING_DWELL_MINUTES` — but be aware this reintroduces flapping risk near boundaries.
- The household power staleness guard assumes the entity's `last_updated` timestamp is exposed by the HA websocket integration. If your entity doesn't expose this, the staleness check will always report stale and set free PV to zero — remove the Change node and the staleness guard in that case.
- Tested on **Node-RED 4.1.10** with **node-red-contrib-home-assistant-websocket 0.80.3**, **Home Assistant OS 18.2 / Core 2026.7.4 / Supervisor 2026.07.5**, and a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region.

---

## Contributing

Shared as a starting point. If you adapt it for a different tariff structure, solar forecast provider, or inverter, feel free to open a PR or issue.

---

## License

MIT — do whatever you like, no warranty implied.
