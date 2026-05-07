# 8. Set up the SolarGo TOU schedule (Method 3 only)

If you're going with **Method 3 (Hybrid)** - which we recommend - the free-window charging is handled by the inverter's firmware, not by Home Assistant. You set up two schedule slots once in the SolarGo app (one for the free-window charge, one for the peak-window discharge) and the inverter handles both windows every day without HA needing to lift a finger for the inverter side. HA's only job is the smart layer at peak time: SOC guard, grid-export limit, profit notifications.

This is the bit of Method 3 that makes the throughput case work - the inverter combines grid AC and solar DC together to charge at up to 13.5kW on the 10kW model, which HA can't ask for. The 13.5kW number is from GoodWe's own [ESA Series datasheet, V2.1 April 2026](https://admin.goodwe.com/Api/downloadFile?id=3448&mid=60&type=2) - "Max. Charging Power" row for the GW9.999K-EHA-G20. Smaller ESA models have proportionally lower ceilings (5kW model = 7.5kW, 8kW model = 12kW). Community confirmation of the same effect: Whirlpool thread "Goodwe ESA maximum charge rate?" by user nutttr with confirmation from Zerosignal (<https://forums.whirlpool.net.au/thread/9kppp8k2>).

If you're using **Method 1 or Method 2**, skip this guide. You don't need TOU schedules because HA's controlling the inverter directly.

## SolarGo vs SEMS+

GoodWe has two apps:

- **SolarGo** - the consumer app. Most settings end users need are in here.
- **SEMS+ Pro** - the installer/dealer portal. More advanced settings, but locked behind an installer login.

For TOU setup, **SolarGo is what you want**. Some older guides reference SEMS+ for this, but on current SolarGo versions it's all in the consumer app.

## Step 1 - Open SolarGo and navigate to your inverter

Open the SolarGo app, log in, and tap your plant / inverter. You should land on a dashboard showing live PV, battery, grid, and load.

Tap the settings cog / gear icon to enter the inverter's settings.

## Step 2 - Find the TOU section

In current SolarGo versions, look for **"TOU"** or **"Time of Use"** under the inverter settings menu. Older app versions and some firmware revisions used the label "Economic Mode" for the same thing - if your app shows that instead of "TOU", it's the same feature. The Glossary covers the naming history if you're curious.

If you can't find it: the menu may be under "Advanced" or require an installer password. The Whirlpool threads ["Home Assistant setup with GoodWe inverter"](https://forums.whirlpool.net.au/thread/9xv6wp84) and ["GoodWe ESA - Setting export TOU with SOC limit"](https://forums.whirlpool.net.au/thread/9n111qlk) are the best places to ask if you're stuck.

## Step 3 - Make sure the inverter is in TOU mode

Before TOU schedules take effect, the inverter has to be in TOU / Economic mode (the schedule-based operation mode). Some app versions require you to explicitly select it; others let you configure schedules from any mode and apply them when you switch.

Set the active working mode to **TOU** (or **Economic Mode** if your app uses that wording). If the app warns about overwriting other settings, that's expected - this is the only mode that respects TOU schedules.

## Step 4 - Create the charge slot (free window)

Add a new schedule entry:

| Setting | Value |
|---|---|
| **Start time** | `11:00` |
| **End time** | `14:00` |
| **Action / Mode** | **Charge** |
| **SOC target** | `100%` (charge to full) |
| **Power source** | **Grid** (high priority) - or "Grid + PV" if your version separates them |
| **Days** | All days (Mon-Sun) |
| **Enabled** | Yes |

Save the slot.

> **Why these specific settings?**
> - **11:00-14:00** matches the GloBird Zero Hero free window. If you're on a different free-window plan (some VPP variants are 10-13 or noon-3), adjust to match.
> - **Charge to 100%** because LFP chemistry tolerates regular full charges and the BMS calibrates itself at the top end. Charging to less than 100% leaves money on the table during peak. (See the strategy guide's "Free window" section for detail and citation.)
> - **Grid priority** so the inverter prefers grid power for the charge (which is free during this window) and uses solar as a top-up. The opposite priority would consume your solar first and only top up from grid if solar wasn't enough - fine, but you'd export less.

## Step 5 - Create the discharge slot (peak window)

Add a second schedule entry. This one tells the inverter "during peak, you're allowed to discharge at full inverter output - HA will manage how much actually goes to the grid via the export-limit entity":

| Setting | Value |
|---|---|
| **Start time** | `18:00` |
| **End time** | `21:00` (or `20:00` if you're on the older GloBird plan) |
| **Action / Mode** | **Discharge** |
| **Power** | `100%` of inverter output |
| **SOC target** | A low number such as `10%` (HA's SOC guard at 17:56 is the actual floor) |
| **Days** | All days (Mon-Sun) |
| **Enabled** | Yes |

Save the slot.

> **Critical concept: "discharge power" in SolarGo means *total inverter output*, not grid-export specifically.** A 10% setting on a 10kW inverter means 1kW total (house load + grid combined), not 1kW to the grid. That's why we set discharge power to 100% here and let HA's `number.goodwe_grid_export_limit` (set to 5000W in the YAML) be the precise grid-export lever. The inverter's full output covers house load *plus* up to 5kW to the grid.

> **Why a discharge slot at all?** Without it, the inverter is in self-consumption during peak. That works - the battery covers house load and the export limit caps grid export - but the inverter's output is then effectively capped at house demand. With the discharge slot at 100%, the inverter is willing to push house-load + 5kW to the grid in parallel, which is what you actually want during a peak window.

## Step 6 - Confirm timing doesn't collide with HA triggers

Method 3's HA automation has its own time triggers. Here's the full picture across one Zero Hero day:

| Time | What fires | Who runs it |
|---|---|---|
| 00:01 | Midnight reset (export tracker zeroed, fast-charge switch turned off as safety) | HA |
| 11:00 | Charge TOU slot starts (battery charges to 100% from grid + solar blended) | Inverter firmware |
| 14:00 | Charge TOU slot ends | Inverter firmware |
| 17:56 | Pre-peak guard check (HA reads SOC, decides to arm export limit or block) | HA |
| 18:00 | Discharge TOU slot starts (inverter willing to discharge at full output) | Inverter firmware |
| 21:00 | Discharge TOU slot ends | Inverter firmware |
| 21:01 | HA peak_end (export limit restored, profit notification fires) | HA |

Tight, no overlaps, each system owns its own time slots. The 4-minute gap between 17:56 (HA arming) and 18:00 (TOU discharge starting) gives the inverter a moment to be ready before HA's export-limit value matters. The 1-minute gap between 21:00 (TOU discharge ends) and 21:01 (HA restores normal export limit) is the inverse - lets the inverter return to self-consumption before HA opens the export limit back up.

If you're on the older 18:00-20:00 plan, change the discharge slot end time to 20:00 *and* the HA `peak_end` trigger time to 20:01 in the YAML.

## Step 7 - Verify it took effect

The next free window (11:00 the following day, or right now if you're inside one), check the inverter's behaviour:

- Battery SOC should rise toward 100% throughout the window.
- Grid import should be substantial (5-13kW depending on your inverter, battery state, and current solar).
- Once SOC hits 100%, charging stops and the inverter holds - it does **not** drop into self-consumption and start discharging.

The next peak window:

- At 17:56 you should see a "Zero Hero Armed" notification from HA (assuming `input_boolean.zero_hero_enabled` is on and SOC is above your guard threshold).
- During peak, grid export should sit at around 5kW plus whatever house load is. SOC should drop steadily.
- At 21:01 you should see a "Zero Hero Complete" notification with the night's profit calculation.

If the charge slot didn't fire - battery isn't charging during the window - check:

- The inverter's working mode is actually set to TOU (Step 3).
- The slot is enabled.
- The inverter clock is correct. Run the [`goodwe_time_sync.yaml`](../automations/goodwe_time_sync.yaml) automation manually from HA, or check the time on the inverter's local web interface, to make sure it hasn't drifted.

## Critical: do not run Methods 1 or 2 alongside this

We say this in the strategy README and we'll say it again here. Method 1 (and any operation-mode change in Method 2) **deletes both your TOU slots** ([Whirlpool thread "GoodWe ESA - Setting export TOU with SOC limit"](https://forums.whirlpool.net.au/thread/9n111qlk)). The first time HA writes "Eco mode" or "General mode" to `select.goodwe_inverter_operation_mode`, your carefully-set TOU configuration is gone.

If you're running Method 3, your YAML in HA should never touch the operation mode - and the [Method 3 automation in this repo](../../automations/globird/method3_hybrid/globird_zero_hero.yaml) is designed exactly that way. Don't add additional automations that change `select.goodwe_inverter_operation_mode` while you're on Method 3.

## What you should have at the end of this guide

- SolarGo set to TOU mode.
- A daily 11:00-14:00 **charge** slot, target 100% SOC, grid priority.
- A daily 18:00-21:00 (or 18:00-20:00) **discharge** slot at 100% inverter power.
- Verified both slots fire (watch battery SOC climb during free, watch grid export during peak).
- No HA automations touching operation mode.

Next: [First-run checklist](./09_first_run_checklist.md).
