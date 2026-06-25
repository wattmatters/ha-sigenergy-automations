# Battery Control Ownership State Machine

A coordination layer **shared as an example** that eliminates a class of handover bugs in the Sigenergy automation collection by introducing a single source of truth for which automation currently has the right to control Remote EMS.

This is posted as a reference to adapt to your own system — not a plug-and-play solution.

---

## Why this exists

The original design had each automation (Peak Export, Supplemental Charging, Bias Offset) independently *infer* whether it currently "owned" Remote EMS by checking things like:
- Is the Remote EMS switch on or off?
- Is the operating mode select in Command Discharging?
- Is the export limit at a particular value?

This worked, but every automation was guessing at shared state from secondhand signals, and every guess was a place two automations could disagree — especially right at time-window boundaries, where one automation's "outside my window, clean up" branch could race against another automation's "inside my window, take over" branch. Several of the bugs documented in this collection's automation READMEs (flapping Remote EMS, post-window cleanup colliding with the next owner, mode-equality checks proving unreliable) all trace back to this same root cause: there was no single place that recorded *whose turn it currently was*.

The state machine fixes this by introducing exactly one thing that decides ownership, and having every consumer automation gate on it.

---

## Architecture

```
+----------------------------------------------------------+
| Battery Control Ownership State Machine (this folder)    |
|                                                          |
| Runs every minute + on AEMO override change.             |
| Writes input_select.battery_control_owner ONLY.          |
| Never touches Remote EMS, mode select, or limits.        |
|                                                          |
| Priority (highest first):                                |
|   1. aemo_override          (AEMO critical event active) |
|   2. supplemental_charging  (cheap/free rate window)     |
|   3. peak_export            (peak feed-in window)        |
|   4. bias_offset            (everything else - default)  |
|   5. none                   (safety-net, unreachable)    |
+----------------------------------------------------------+
                              |
                              | input_select.battery_control_owner
                              v
          +-------------------+-------------------+
          |                   |                   |
          v                   v                   v
 +----------------+  +----------------+  +----------------+
 | Peak Export    |  | Supplemental   |  | Bias Offset    |
 | Control        |  | Grid Charging  |  | Export         |
 | (State         |  | (State         |  | (State         |
 |  Machine)      |  |  Machine)      |  |  Machine)      |
 |                |  |                |  |                |
 | Gate: owner    |  | Gate: owner == |  | Gate: owner == |
 | == 'peak_      |  | 'supplemental_ |  | 'bias_offset'  |
 |  export'       |  |  charging'     |  |                |
 +----------------+  +----------------+  +----------------+

(AEMO Critical Event Export Control operates independently and
 does not read battery_control_owner -- see note below)
```

Each consumer automation's **first condition, on every trigger**, is the ownership gate: if `battery_control_owner` doesn't match its name, it does nothing at all — not even a baseline reset. This means:

- There is no need for "outside my window, clean up" branches — being outside your window means the gate simply doesn't pass, so nothing runs
- A bug in one automation's internal logic can't cause it to act outside its slot, because the gate is checked before anything else
- Adding a new consumer automation means adding its claim to the arbitrator's priority list — there's exactly one place to look to understand the whole system's priority order

### Note on the AEMO automation

The AEMO Critical Event Export Control automation does **not** read `battery_control_owner` — it continues to operate independently, exactly as before. However, the state machine arbitrator *does* read the same flag the AEMO automation sets (`input_boolean.aemo_critical_override_active`) and gives it top priority, so the other three consumer automations correctly stand aside while AEMO is active. If you use the AEMO automation from this collection, no changes are needed to it.

### Note on EV DC Charger Cooldown

The EV DC Charger Stop Cooldown Timer automation is also unchanged and does not read or write `battery_control_owner`. It simply starts a timer when EV charging stops; the Bias Offset (State Machine) automation checks that timer's state as one of its own RELEASE conditions, the same as before.

---

## The "baseline reset" principle

A pattern used consistently across all three consumer automations: **every branch, regardless of its primary purpose, unconditionally resets the grid export limit and charging limit to safe maximums on entry** (state-guarded only for the switch/mode, not for the numeric limits). 

This matters because none of the three consumer automations exclusively owns the export limit or charging limit numbers — any of them could leave a stale low value in place for whichever automation takes over next. For example, if Supplemental Charging's self-consumption branch doesn't reset the export limit, and the previous owner (Bias Offset) had set it to a tiny value, genuine excess solar later in the window would be wrongly capped.

If you add a new consumer automation, give it the same treatment: reset shared numeric limits to safe values on every cycle it runs, even if that particular automation doesn't use them.

---

## Requirements

### Helper entity

Create this `input_select` helper:

| Entity | Type | Options |
|---|---|---|
| `input_select.battery_control_owner` | Select | `none`, `aemo_override`, `supplemental_charging`, `peak_export`, `bias_offset` |

### Dependent automations

This state machine is only useful alongside the state-machine-gated versions of the consumer automations:

| Automation | Reads owner state |
|---|---|
| [Peak Export Control (State Machine)](../peak-export-control-state-machine/) | `peak_export` |
| [Supplemental Grid Charging (State Machine)](../supplemental-grid-charging-state-machine/) | `supplemental_charging` |
| [Grid Bias Offset Export (State Machine)](../grid-bias-offset-export-state-machine/) | `bias_offset` |
| AEMO Critical Event Export Control | *(does not read it — operates independently)* |
| EV DC Charger Stop Cooldown Timer | *(does not read it — unchanged)* |

The original (non-state-machine) versions of Peak Export, Supplemental Charging, and Bias Offset have been moved to the [archive](../archive/) folder — see that folder's README for why they were superseded and what bugs the new design fixes.

---

## Installation

1. Create the `input_select.battery_control_owner` helper with the five options listed above.

2. Copy this automation's YAML to your `config/automations/` directory.

3. Find and replace all `← REPLACE` and `← ADJUST` markers — primarily the time windows and the AEMO override entity, if you use a different name.

4. Install the three state-machine-gated consumer automations (see their individual folders).

5. **Important**: disable (don't necessarily delete) the original non-state-machine versions of Peak Export, Supplemental Charging, and Bias Offset before enabling the new ones, to avoid two automations fighting over the same hardware. Keeping the originals disabled-but-present gives you a fallback if you need to revert.

6. Reload automations or restart Home Assistant.

---

## Adapting the priority order

If your system has different time windows or additional automations, the arbitrator's `choose` block is the single place to express priority. A few principles worth keeping if you adapt this:

- **Mirror the exact live signal**, don't re-derive an approximation. The arbitrator's AEMO branch checks the same boolean the AEMO automation itself sets after taking action — not a raw price-threshold input that might not yet have triggered real activity.
- **Order matters**: list higher-priority claims first. The last non-default branch should be your "default" automation — the one that's happy to run at any time not claimed by something more specific.
- **Keep a true safety-net default** (`none`) even if you believe it's unreachable. It costs nothing and means a future change that narrows another branch's scope fails loudly (nothing happens) rather than silently (the wrong automation takes over).

---

## Known limitations / notes

- The arbitrator polls every minute plus on AEMO state changes. If your time windows are very short or you need faster handover, consider adding more state triggers (as the bias offset and peak export consumer automations do).
- This pattern assumes a small, fixed set of mutually exclusive consumers. If your system has automations that need to run *concurrently* (not just take turns), this single-owner model doesn't fit — you'd need a different coordination primitive (e.g. a set of independent flags rather than one selector).
- Tested on **Home Assistant OS 17.3 / Core 2026.5.2 / Supervisor 2026.05.0** with a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region.

---

## Contributing

Shared as a starting point. If you adapt this coordination pattern for a different set of automations or a different inverter platform, feel free to open a PR or issue to share what you changed.

---

## License

MIT — do whatever you like, no warranty implied.
