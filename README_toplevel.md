# ha-sigenergy-automations

A collection of Home Assistant automations for **Sigenergy energy controller owners** in the Australian NEM, shared as examples for adaptation to your own system.

These automations were developed and tested on a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region, running **Home Assistant OS 17.3 / Core 2026.5.2 / Supervisor 2026.05.0** with the [sigenergy2mqtt](https://github.com/seud0nym/home-assistant-addons/tree/main/sigenergy2mqtt) add-on integration.

> **Important:** These are not plug-and-play solutions. They are posted as working examples to help others understand the approach and adapt it to their own system, tariff, and configuration. You will need to update entity IDs, time windows, thresholds, and helper references to suit your setup.

---

## Automations

| Automation | Description |
|---|---|
| [AEMO Critical Event Export Control](./aemo-critical-event-export/) | Maximises battery export during AEMO critical price events |
| [Peak Export Control](./peak-export-control/) | Manages battery export during the evening peak feed-in bonus period, with Zero Hero protection |
| [Supplemental Grid Charging](./supplemental-grid-charging/) | *(coming soon)* |
| [EV DC Charger Stop Cooldown Timer](./ev-dc-charger-cooldown/) | *(coming soon)* |
| [Grid Bias Offset Export](./grid-bias-offset-export/) | *(coming soon)* |

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
| `input_boolean.battery_export_enabled` | Toggle | Peak Export, AEMO Critical |
| `input_boolean.aemo_critical_override_active` | Toggle | AEMO Critical, Peak Export |
| `input_boolean.aemo_pre_alert` | Toggle | AEMO Critical |
| `input_boolean.aemo_critical_event` | Toggle | AEMO Critical |
| `input_number.aemo_critical_threshold` | Number | AEMO Critical |
| `input_number.aemo_pre_alert_threshold` | Number | AEMO Critical |
| `input_number.aemo_nsw_spot_price` | Number | AEMO Critical |
| `input_number.battery_peak_export_maximum_rate` | Number | Peak Export |
| `input_number.battery_minimum_export_rate_zero_hero` | Number | Peak Export |
| `input_number.battery_minimum_soc_end_of_export` | Number | Peak Export |
| `input_number.battery_export_suggested_rate` | Number *(optional)* | Peak Export |
| `input_text.battery_export_strategy` | Text *(optional)* | Peak Export |
| `input_number.battery_charge_suggested_rate` | Number *(optional)* | AEMO Critical |

See each automation's individual README for the full list of helpers it requires and how they are used.

---

## How the automations interact

These automations are designed to work together without conflicting:

- **AEMO Critical Event** takes priority over everything — when active, it sets `input_boolean.aemo_critical_override_active` to `on`, which suppresses the Peak Export automation
- **Peak Export** manages the evening export window and respects the AEMO override flag
- Other automations (charging, cooldown, bias offset) are designed to avoid interfering with active export or override states

Each automation's README describes its interaction points with the others.

---

## Contributing

Shared in the spirit of giving others a starting point. If you adapt any of these for a different inverter, NEM region, or tariff structure, feel free to open a PR or issue to share what you changed — it could save the next person a lot of time.

---

## License

MIT — do whatever you like, no warranty implied.
