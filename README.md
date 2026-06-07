# ha-sigenergy-automations

A collection of Home Assistant automations for **Sigenergy energy controller owners** in the Australian NEM, shared as examples for adaptation to your own system.

These automations were developed and tested on a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region, running **Home Assistant OS 17.3 / Core 2026.5.2 / Supervisor 2026.05.0** with the [sigenergy2mqtt](https://github.com/seud0nym/home-assistant-addons/tree/main/sigenergy2mqtt) add-on integration.

> **Important:** These are not plug-and-play solutions. They are posted as working examples to help others understand the approach and adapt it to their own system, tariff, and configuration. You will need to update entity IDs, time windows, thresholds, and helper references to suit your setup.

---

## Acknowledgements

These automations were developed with assistance from [Claude](https://claude.ai), Anthropic's AI assistant, which helped with automation logic, anonymisation, and documentation.

---

## Automations

| Automation | Description |
|---|---|
| [AEMO Critical Event Export Control](./aemo-critical-event-export/) | Maximises battery export during AEMO critical price events |
| [Peak Export Control](./peak-export-control/) | Manages battery export during the evening peak feed-in bonus period, with Zero Hero protection |
| [Supplemental Grid Charging](./supplemental-grid-charging/) | Manages battery charging during a cheap or free electricity rate period |
| [EV DC Charger Stop Cooldown Timer](./ev-dc-charger-cooldown/) | Starts a cooldown timer when DC charging stops, preventing other automations from acting before load sensors settle |
| [Grid Bias Offset Export](./grid-bias-offset-export/) | Applies a small persistent export offset to eliminate metering bias imports the inverter cannot detect |

---

## Node-RED Flows

Two Node-RED flows act as the decision-making layer for the charging and export automations. They run calculations every minute using solar forecasts, battery state, and household consumption, and write their outputs to HA helper entities that the automations above then act on.

| Flow | Description |
|---|---|
| [Battery Charging Calculator](./node-red/battery-charging-calculator/) | Calculates whether and how fast to charge during the cheap/free rate period |
| [Battery Export Calculator](./node-red/battery-export-calculator/) | Calculates whether and how fast to export during the evening peak feed-in period |

See the [Node-RED README](./node-red/) for prerequisites, installation instructions, and how the flows connect to the HA automations.

### Supporting automation

The Battery Export Calculator reads `input_boolean.aircon_expected_overnight` as a cross-check, but the primary connection to battery management is more direct: the overnight load prediction automation calculates and writes a dynamic SOC floor to `input_number.battery_minimum_soc_end_of_export` each afternoon, which the Peak Export Control automation uses to limit how aggressively the battery discharges during the peak window. See the [ha-overnight-load-prediction](https://github.com/wattmatters/ha-overnight-load-prediction) repository for the full implementation — the same pattern can be adapted for EV charging, hot water, or any other significant overnight load.

---

## Common Requirements

All automations in this collection share the following prerequisites.

### Sigenergy Integration

There are two common Home Assistant integrations for Sigenergy inverters:

- **[sigenergy2mqtt](https://github.com/seud0nym/home-assistant-addons/tree/main/sigenergy2mqtt)** — MQTT-based add-on. Entity naming in these automations follows this integration's conventions.
- **[Sigenergy Local Modbus](https://github.com/TypQxQ/Sigenergy-Local-Modbus)** — HACS integration using local Modbus polling. Entity names may differ; use Developer Tools → States to find your equivalents.

### AEMO NEM Web Integration

Price data is sourced from the [AEMO NEM Web](https://github.com/pvandenh/aemo_nemweb) HACS integration (used by the AEMO Critical Event automation). Install via HACS and configure for your NEM region (`nsw1`, `vic1`, `sa1`, `qld1`, etc.).

### Shared Helper Entities

Several helpers are referenced across multiple automations. Create these once in **Settings → Devices & Services → Helpers**:

| Entity | Type | Used by |
|---|---|---|
| `input_boolean.battery_export_enabled` | Toggle | Peak Export, AEMO Critical, Export Calculator (NR) |
| `input_boolean.aemo_critical_override_active` | Toggle | AEMO Critical, Peak Export |
| `input_boolean.aemo_pre_alert` | Toggle | AEMO Critical |
| `input_boolean.aemo_critical_event` | Toggle | AEMO Critical |
| `input_number.aemo_critical_threshold` | Number | AEMO Critical |
| `input_number.aemo_pre_alert_threshold` | Number | AEMO Critical |
| `input_number.aemo_nsw_spot_price` | Number | AEMO Critical |
| `input_number.battery_peak_export_maximum_rate` | Number | Peak Export, Export Calculator (NR) |
| `input_number.battery_minimum_export_rate_zero_hero` | Number | Peak Export |
| `input_number.battery_minimum_soc_end_of_export` | Number | Peak Export |
| `input_number.battery_export_suggested_rate` | Number | Peak Export ← set by Export Calculator (NR) |
| `input_text.battery_export_strategy` | Text | Informational ← set by Export Calculator (NR) |
| `input_text.battery_export_urgency` | Text | Informational ← set by Export Calculator (NR) |
| `input_number.battery_charge_suggested_rate` | Number | AEMO Critical, Supplemental Charging ← set by Charging Calculator (NR) |
| `input_text.battery_charge_strategy` | Text | Informational ← set by Charging Calculator (NR) |
| `input_text.battery_charge_urgency` | Text | Informational ← set by Charging Calculator (NR) |
| `input_boolean.battery_supplemental_charging_required` | Toggle | Supplemental Charging ← set by Charging Calculator (NR) |
| `timer.ev_dc_charger_stop_cooldown` | Timer | EV Cooldown, Grid Bias Offset |
| `input_number.grid_bias_offset_export_rate` | Number | Grid Bias Offset |
| `input_boolean.grid_bias_offset_active` | Toggle | Grid Bias Offset |
| `input_boolean.managed_period_active` | Toggle | Grid Bias Offset |
| `input_boolean.aircon_expected_overnight` | Toggle | Export Calculator (NR) ← set by overnight load prediction automation |
| `input_number.overnight_soc_boost` | Number | Overnight load prediction — manual top-up (0–30%, default 0%) |
| `sensor.ewma_grid_active_power` | Filter helper | Grid Bias Offset — see that automation's README for setup |

See each automation's individual README for the full list of helpers it requires and how they are used.

---

## How everything fits together

The Node-RED flows and HA automations are designed to work as a layered system without conflicting:

- **Node-RED flows** run every minute and write strategy decisions to helper entities — they decide *whether* and *how much* to charge or export
- **AEMO Critical Event** takes priority over everything — when active, it sets `input_boolean.aemo_critical_override_active` to `on`, which suppresses Peak Export and other automations
- **Peak Export** manages the evening export window, reads the suggested rate from the Export Calculator, and respects the AEMO override flag
- **Supplemental Grid Charging** manages the cheap-rate charging window, reads the suggested rate from the Charging Calculator, and respects the AEMO override flag
- **EV DC Charger Cooldown** and **Grid Bias Offset** operate outside managed windows and both respect the AEMO override flag
- **Overnight Load Prediction** (separate repo) runs each afternoon and sets the aircon flag consumed by the Export Calculator

Each automation's README describes its specific interaction points with the others.

---

## Contributing

Shared in the spirit of giving others a starting point. If you adapt any of these for a different inverter, NEM region, or tariff structure, feel free to open a PR or issue to share what you changed — it could save the next person a lot of time.

---

## License

MIT — do whatever you like, no warranty implied.
