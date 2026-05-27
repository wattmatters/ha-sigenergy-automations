# Battery Export Calculator

A Node-RED flow **shared as an example** that calculates an optimal battery export strategy for an evening peak feed-in period, balancing maximum export against overnight battery reserve requirements.

This is posted as a reference to adapt to your own system — not a plug-and-play solution. Time windows, reserve values, and export caps are all specific to the author's system and must be adjusted.

---

## What it does

Polls seven Home Assistant sensors every 60 seconds, joins them, rate-limits to one calculation per minute, and passes the data to a JavaScript function that calculates an export strategy. The result is written back to four HA helper entities consumed by the [Peak Export Control](../../peak-export-control/) automation.

### Export modes

| Mode | Description |
|---|---|
| `ACTIVE_EXPORT` | In export period with sufficient capacity — export at calculated rate |
| `ZERO_HERO_PROTECTION` | Capacity marginal but above minimum SOC — export at minimum rate to maintain feed-in bonus |
| `BATTERY_TOO_LOW` | Battery below minimum SOC — export disabled to protect overnight reserve |
| `PLANNED_EXPORT` | Before export period — forecasting that export will be possible |
| `PLANNED_NO_EXPORT` | Before export period — forecasting insufficient capacity |
| `EXPORT_PERIOD_ENDED` | After export period — normal operation |

### Zero Hero / feed-in bonus protection

During the export period, the flow prioritises maintaining a minimum SOC (`ZERO_HERO_MIN_BATTERY_SOC`) to preserve eligibility for any feed-in bonus or zero-export incentive on your tariff. Even if the calculated net exportable energy is negative, export is maintained at a minimum rate (0.5 kW) as long as the battery is above this threshold.

### Overnight reserve

The flow calculates a dynamic overnight reserve based on:
- A base overnight reserve (kWh)
- An additional reserve if overnight aircon is expected (read from `input_boolean.aircon_expected_overnight`)
- A safety buffer

Only energy above this reserve is available for export. See the [ha-overnight-load-prediction](https://github.com/wattmatters/ha-overnight-load-prediction) repository for an example of how to automate the `aircon_expected_overnight` flag.

---

## Configuration

Open the **Calculate Export Strategy** function node and edit the constants at the top:

| Constant | Default | Description |
|---|---|---|
| `EXPORT_PERIOD_START_HOUR` | `18` | ← REPLACE: peak export start hour (24h) |
| `EXPORT_PERIOD_END_HOUR` | `20` | ← REPLACE: peak export end hour (24h) |
| `EXPORT_PERIOD_DURATION` | `2` | ← REPLACE: export period duration in hours |
| `MAX_EXPORT_CAP_KWH` | `10` | ← REPLACE: your retailer's export cap kWh (remove if no cap) |
| `BASE_OVERNIGHT_RESERVE_KWH` | `8` | ← REPLACE: base overnight battery reserve kWh |
| `AIRCON_ADDITIONAL_RESERVE_KWH` | `8` | ← REPLACE: additional reserve if aircon expected kWh |
| `SAFETY_BUFFER_KWH` | `1` | Safety buffer added to reserve |
| `MIN_SOC_PERCENT` | `20` | ← REPLACE: absolute minimum battery SOC % |
| `ZERO_HERO_MIN_BATTERY_SOC` | `25` | ← REPLACE: minimum SOC for feed-in bonus eligibility % |

---

## Input sensors

Replace all `YOUR_` entity IDs in the poll-state nodes with your own:

| Topic | Node name | Entity to use |
|---|---|---|
| `battery_soc` | Battery SOC % | Your battery SOC sensor (e.g. `sensor.sigen_0_inverter_1_battery_soc`) |
| `battery_available_discharge` | Battery Available Discharge Energy | Your available discharge energy sensor (e.g. `sensor.sigen_0_inverter_1_available_battery_discharge_energy`) |
| `battery_capacity` | Battery Total Capacity kWh | `sensor.sigen_0_inverter_1_rated_battery_capacity` or `input_number.battery_total_capacity` |
| `solar_remaining` | Solar Forecast Remaining Today | `sensor.solcast_pv_forecast_forecast_remaining_today` |
| `household_power` | Current Household Power | Your total consumed power sensor (W) |
| `aircon_overnight` | Aircon Expected Overnight | `input_boolean.aircon_expected_overnight` |
| `max_export_rate` | Peak Export Maximum Rate | `input_number.battery_peak_export_maximum_rate` |

**Note on join node:** The join node is configured to wait for **7 inputs**. If you add or remove sensors, adjust the **Count** field in the join node accordingly.

---

## Output helpers

The flow writes to these HA helpers (create them before deploying):

| Entity | Type | Consumer |
|---|---|---|
| `input_boolean.battery_export_enabled` | Toggle | Peak Export Control automation |
| `input_number.battery_export_suggested_rate` | Number | Peak Export Control automation |
| `input_text.battery_export_strategy` | Text | Informational / dashboard display |
| `input_text.battery_export_urgency` | Text | Informational / dashboard display |

---

## Known limitations / notes

- The overnight reserve constants (`BASE_OVERNIGHT_RESERVE_KWH`, `AIRCON_ADDITIONAL_RESERVE_KWH`) are fixed values specific to the author's household. Consider making these `input_number` helpers if you want to adjust them without editing the flow.
- The `MAX_EXPORT_CAP_KWH` assumes your retailer caps export credits at 10 kWh per period. Remove or adjust if your tariff has no cap or a different cap.
- The `ZERO_HERO_MIN_BATTERY_SOC` of 25% assumes a ~32 kWh battery (25% ≈ 8 kWh). Scale this to your battery size and overnight consumption.
- The rate limiter node prevents the calculation running more than once per minute even if multiple sensors update simultaneously.
- Tested on **Node-RED 4.1.10** with **node-red-contrib-home-assistant-websocket 0.80.3**, **Home Assistant OS 17.3 / Core 2026.5.2**, and a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region.

---

## Contributing

Shared as a starting point. If you adapt it for a different tariff structure, export cap, or overnight load model, feel free to open a PR or issue.

---

## License

MIT — do whatever you like, no warranty implied.
