# Method 1: App-only (no Home Assistant)

> Before you copy anything: read the [strategy guide](../) to understand why Methods 2-4 add HA on top of this one. If you're unsure what "TOU" or "Economic Mode" means, see the [Glossary](../../../GLOSSARY.md).

The simplest possible Zero Hero setup. Everything happens in the GoodWe SEMS+ app (or SolarGo if that's what you've already got working). No Home Assistant, no HACS, no helpers, no YAML. The inverter's firmware does the work; you check the results in the app.

This is the right starting point if any of the following apply:

- You haven't set up Home Assistant yet and want a working baseline before adding it.
- You'd rather not run extra software just for energy automation.
- You want to understand what the inverter does on its own before deciding whether HA is worth the effort.
- You want a reference point to compare the HA methods against.

If you already know you want HA, skip this and go to [Method 4 (Hybrid)](../method3_hybrid/) - the recommendation. But come back here if you want a clean explanation of how TOU works at the inverter level, since the HA methods all build on top of it.

## What this method does

Two TOU schedule slots in SEMS+:

- **Charge slot:** 11:00-14:00. The inverter charges the battery to 100% from the grid (free during the Zero Hero window) and from any solar that's available.
- **Discharge slot:** 18:00-21:00 (or 18:00-20:00 on older Zero Hero plans). The inverter discharges the battery into your house and out to the grid, capped at a power level you set, with a SOC floor so it doesn't drain to zero.

That's the whole automation. Outside those windows the inverter sits in self-consumption (use solar first, fill from battery, import from grid only if needed).

## What you need

- A GoodWe ESA inverter wired up and online.
- The SEMS+ app on your phone, logged in to your GoodWe account. SolarGo also works for this if you're already using it; we recommend SEMS+ for new setups since GoodWe is migrating consumer features there.
- A GloBird Zero Hero plan (or similar dynamic plan with fixed free + peak windows).
- That's it. No Home Assistant, no integrations, no helpers, no YAML.

## Setup

The full step-by-step including SEMS+ menu navigation lives at [prerequisites/08_sems_tou_schedule.md](../../../prerequisites/08_sems_tou_schedule.md). The summary:

### Step 1 - Switch the inverter to TOU / Economic mode

In SEMS+, navigate to your inverter's settings and set the working mode to **TOU** (called "Economic Mode" on older app versions - same thing). This is the only mode that respects time-window schedules.

### Step 2 - Create the charge slot

| Setting | Value |
|---|---|
| **Start time** | `11:00` |
| **End time** | `14:00` |
| **Mode** | **Charge** |
| **SOC target** | `100%` (or `90%` if you want to stay strictly within GoodWe's 10-90% warranty recommendation - see the LFP note in the [strategy guide](../#how-the-windows-work)) |
| **Power source** | **Grid** (high priority) |
| **Days** | All days |
| **Enabled** | Yes |

### Step 3 - Create the discharge slot

| Setting | Value |
|---|---|
| **Start time** | `18:00` |
| **End time** | `21:00` (or `20:00` on older Zero Hero plans) |
| **Mode** | **Discharge** |
| **Power** | `5kW` is a reasonable starting point for the 10kW model on Zero Hero - this caps the total inverter output (house + grid combined). See the discharge-power note below. |
| **SOC target** | `30%` is a reasonable starting floor - the inverter won't discharge below this. Tune up if you find yourself running short overnight, down if you wake up with charge to spare. |
| **Days** | All days |
| **Enabled** | Yes |

### Step 4 - Verify it works

Watch the next free window:

- Battery SOC climbs toward 100% (or 90% if that's your target) between 11:00 and 14:00.
- Once SOC hits the target, charging stops.

Watch the next peak window:

- Battery SOC drops as the inverter discharges into the house and grid.
- Discharge stops when SOC hits your floor.

That's it. The inverter handles the rest day after day.

## The discharge-power gotcha (read this once)

In SEMS+'s TOU discharge slot, **"discharge power" means total inverter output, not grid export specifically**.

Worked example: you set discharge power to 5kW. Your house is drawing 2kW. The inverter discharges up to 5kW total - 2kW covers the house, 3kW goes to the grid. If your house load spikes to 5kW (someone runs the dryer and the kettle), all 5kW goes to the house and nothing reaches the grid. The total figure is the inverter's output ceiling, not a guaranteed grid export.

This is the imprecision that the HA methods (especially Method 4) fix. With HA you can pin grid export to a specific number regardless of house load. Without HA, your grid export drifts with whatever's plugged in.

## What you're missing without HA

This method gets you the basic charge-during-free / discharge-during-peak cycle, which is most of the value. But there are real things HA adds that the app alone can't:

- **Precise grid-export control during peak.** SEMS+ pins total inverter output; HA's `number.goodwe_grid_export_limit` pins grid export specifically. With Zero Hero's "first 10kWh at 15c, rest at 6c" structure, hitting the cap precisely matters.
- **Dynamic SOC guard.** SEMS+ will discharge to whatever SOC floor you set even if conditions change (cloudy day, free-window charge didn't fill the battery, etc.). HA can decide "today's not the day, skip peak export" based on live SOC at 17:56.
- **Profit notifications.** SEMS+ shows you what charged and discharged but doesn't compute "you exported X kWh tonight at the super rate, plus the daily credit, total $Y." HA does that and pushes it to your phone each evening.
- **Helper-tunable rates.** When GloBird adjusts the super rate or daily credit (it happens), updating SEMS+ tariff configuration is fiddly. With HA the rates live in number helpers you can edit from the dashboard in two clicks.

If those things matter to you, jump to:

- [Method 2 (Standard Eco Mode)](../method1_standard/) - HA-driven, simpler than Method 4 but with tradeoffs.
- [Method 3 (EMS RAM Commands)](../method2_ems/) - the right tool if you're on a dynamic-pricing VPP like Amber.
- [Method 4 (Hybrid - recommended)](../method3_hybrid/) - keeps this method's TOU schedule and adds HA on top for the smart layer. The default recommendation for fixed-window plans like Zero Hero.

If they don't, this method is fine. You're getting most of the Zero Hero benefit with none of the complexity.

## Watch out for

- **The discharge-power semantics.** Read the gotcha above twice. It's the most common confusion.
- **Inverter clock drift.** GoodWe inverters can drift a few minutes per week. If your clock drifts and your TOU charge slot fires from 11:04 to 14:04 instead of 11:00 to 14:00, you've lost ~7% of the free window. Check the inverter clock against your phone's clock periodically and resync via the app if needed. Methods 2-4 include an HA automation that does this daily.
- **Firmware availability for the 13.5kW combined-charging capability.** On the single-phase 10kW ESA, the firmware that combines grid AC and solar DC for 13.5kW battery charging has been rolling out from around April 2026. Some firmware releases have it, some don't, and some users have had to ask GoodWe Level 2 support for a standalone push. If you're seeing your battery cap at ~10kW during the free window despite plenty of solar, this is the likely cause.
- **Plan rate changes.** GloBird occasionally adjusts the super rate, base rate, or daily credit. Check your latest bill and your plan documents periodically.
- **No safety-net SOC guard.** If your battery is going into peak with limited charge (cloudy day, free window cut short, etc.), this method will discharge down to your set SOC floor and then start importing from the grid at peak rates to cover any remaining demand. There's no "skip peak today" logic. HA methods add that.

## Going further

If this method works for you, brilliant - you're done. If you want any of the things in the "What you're missing" list above, [Method 4 (Hybrid)](../method3_hybrid/) is the natural next step. It keeps the TOU schedules you've just set up and adds HA's smart layer at peak time.
