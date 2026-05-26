# SIGEN – EV DC Charger Stop Cooldown Timer

A Home Assistant automation **shared as an example** for Sigenergy energy controller owners who also have a DC-coupled EV charger and want to prevent other automations from acting immediately after EV charging stops.

This is posted as a reference to adapt to your own system — not as a plug-and-play solution.

---

## What it does

When the DC charger transitions from actively charging to idle (either the running state leaves `Charging` or output power drops to zero), this automation starts a cooldown timer.

The timer is intended to be used as a condition in other automations — for example, the [Grid Bias Offset Export](../grid-bias-offset-export/) automation checks that this timer is not active before exporting overnight, preventing it from triggering immediately after EV charging stops when load sensors may not yet reflect the true steady-state consumption.

This is particularly useful if your system uses a low-pass or smoothing filter on power readings, as these take time to settle after a large load like an EV charger drops off.

---

## Requirements

### Hardware — SIGEN DC charger integration

This automation requires sensors exposed by your Sigenergy DC charger integration:

| Function | Example entity (sigenergy2mqtt) | What to look for |
|---|---|---|
| DC charger running state | `sensor.sigen_0_dc_charger_1_running_state` | A sensor with a `Charging` state when actively charging |
| DC charger output power | `sensor.sigen_0_dc_charger_1_output_power` | Output power in W |

If you have more than one DC charger, you may need to duplicate the trigger condition for each charger or combine them with an `or` condition.

### Helper Entities

Create this in **Settings → Devices & Services → Helpers**:

| Entity | Type | Notes |
|---|---|---|
| `timer.ev_dc_charger_stop_cooldown` | Timer | Set the duration to your preferred cooldown period (e.g. 10 minutes) |

The cooldown duration is configured on the timer helper itself, not in the automation YAML.

---

## Installation

1. Create the `timer.ev_dc_charger_stop_cooldown` helper, setting the duration to your preferred cooldown period.

2. Copy `sigen_ev_dc_charger_stop_cooldown.yaml` to your `config/automations/` directory, or paste into the Automation editor → Edit in YAML.

3. Find and replace all `← REPLACE` markers:

| Placeholder | What to replace with |
|---|---|
| `sensor.sigen_0_dc_charger_1_running_state` | Your DC charger running state sensor |
| `sensor.sigen_0_dc_charger_1_output_power` | Your DC charger output power sensor |
| `timer.ev_dc_charger_stop_cooldown` | Your cooldown timer entity |

4. Reload automations or restart Home Assistant.

---

## How to use the timer in other automations

In any automation that should wait for the cooldown to expire before acting, add a condition like:

```yaml
condition:
  - condition: state
    entity_id: timer.ev_dc_charger_stop_cooldown
    state: "idle"
```

This ensures the automation only proceeds if the cooldown timer is not currently running.

---

## Known limitations / notes

- This automation triggers on the **falling edge** — when charging stops. If the charger briefly pauses and restarts (e.g. during a charging session with multiple phases), the timer will restart each time. Adjust the cooldown duration to suit your charger's behaviour.
- The `Charging` state value is specific to the sigenergy2mqtt integration. If you use Sigenergy Local Modbus, check the actual state value in Developer Tools → States and update the trigger template accordingly.
- Only applicable if you have a DC-coupled EV charger connected to your Sigenergy system. AC-coupled chargers will have different sensors.
- Tested on **Home Assistant OS 17.3 / Core 2026.5.2 / Supervisor 2026.05.0** with a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region, using the **sigenergy2mqtt** add-on integration.

---

## Contributing

Shared as a starting point. If you adapt it for a different charger type or integration, feel free to open a PR or issue to share what you changed.

---

## License

MIT — do whatever you like, no warranty implied.
