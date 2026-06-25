# Archive — Superseded Automations

This folder contains the **original, time-window-inference versions** of three automations that have been superseded by [state-machine-gated versions](../battery-control-state-machine/).

These are kept here for historical reference and because they remain a valid, simpler starting point for anyone who doesn't need the coordination guarantees the state machine provides — for example, a system with only one or two of these automations active, where handover races are less likely.

---

## Archived automations

| Original | Superseded by |
|---|---|
| [Peak Export Control](./peak-export-control/) | [Peak Export Control (State Machine)](../peak-export-control-state-machine/) |
| [Supplemental Grid Charging](./supplemental-grid-charging/) | [Supplemental Grid Charging (State Machine)](../supplemental-grid-charging-state-machine/) |
| [Grid Bias Offset Export](./grid-bias-offset-export/) | [Grid Bias Offset Export (State Machine)](../grid-bias-offset-export-state-machine/) |

The **AEMO Critical Event Export Control** and **EV DC Charger Stop Cooldown Timer** automations were *not* superseded — they continue to operate exactly as documented in their own folders, unchanged by the state machine project.

---

## Why these were superseded

Each of these three automations independently inferred whether it currently "owned" Remote EMS by checking secondhand signals — was the switch on, was the mode select in a particular state, was the export limit at a particular value. This worked most of the time, but every inference was a place two automations could disagree, particularly at time-window boundaries where one automation's cleanup logic could race against another's takeover logic.

Several bugs documented across this collection trace back to this root cause:
- Remote EMS flapping from a hysteresis gap between the Node-RED suggestion layer and the HA executor layer
- Supplemental charging leaving Remote EMS on past its window end
- Peak export's cleanup branch colliding with the next owner just after the window closed
- Shared-ownership inference via mode equality checks proving unreliable across automations

The [Battery Control Ownership State Machine](../battery-control-state-machine/) introduces a single source of truth for ownership, and each consumer automation gates on it as its first condition. See that folder's README for the full architecture and rationale.

---

## Should you use the archived versions or the state-machine versions?

**Use the state-machine versions if:**
- You're running three or more of these automations together
- You've experienced any handover/flapping issues with the originals
- You want a single place to see and adjust priority ordering

**The archived versions may still suit you if:**
- You're only running one of these automations (e.g. just Peak Export, with no Supplemental Charging or Bias Offset) and have no AEMO override to coordinate with
- You want the simplest possible starting point to understand before adding coordination complexity
- You're adapting this collection for a system with fundamentally different ownership semantics, where the state machine pattern doesn't apply cleanly

If in doubt, start with the state-machine versions — the added complexity of the arbitrator is small relative to the bug classes it eliminates once more than one automation is in play.

---

## Live system note

In the author's own system, these original automations were disabled (not deleted) at cutover and kept as a fallback. They are included here in that same spirit — as a reference for understanding how the design evolved, and as a fallback pattern if you need to revert.
