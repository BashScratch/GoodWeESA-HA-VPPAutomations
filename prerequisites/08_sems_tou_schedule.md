# 8. Set up the SolarGo TOU schedule (Method 3 only)

If you're going with **Method 3 (Hybrid)** - which we recommend - the free-window charging is handled by the inverter's firmware, not by Home Assistant. You set up a schedule once in the SolarGo app and the inverter charges to 100% from grid every day during the GloBird free window without HA needing to lift a finger.

This is the bit of Method 3 that makes the throughput case work - the inverter blends grid AC and solar DC together to charge at up to 13.5kW, which HA can't ask for. ([Whirlpool 9kppp8k2](https://forums.whirlpool.net.au/thread/9kppp8k2) explains the mechanism in detail.)

If you're using **Method 1 or Method 2**, skip this guide. You don't need a TOU schedule because HA's controlling the inverter directly.

## SolarGo vs SEMS+

GoodWe has two apps:

- **SolarGo** - the consumer app. Most settings end users need are in here.
- **SEMS+ Pro** - the installer/dealer portal. More advanced settings, but locked behind an installer login.

For setting up the TOU schedule, **SolarGo is what you want**. Some older guides reference SEMS+ for this, but on current SolarGo versions it's available in the consumer app.

## Step 1 - Open SolarGo and navigate to your inverter

Open the SolarGo app, log in, and tap your plant / inverter. You should land on a dashboard showing live PV, battery, grid, and load.

Tap the settings cog / gear icon to enter the inverter's settings.

## Step 2 - Find the Economic Mode / TOU section

The wording varies by app version and inverter model. Look for one of:

- **"Working Mode"** > **"Economic Mode"**
- **"Operation Mode"** > **"TOU"** or **"TOU Settings"**
- **"Battery Settings"** > **"Charge Schedule"** or **"Time of Use"**
- **"Mode Selection"** > **"Economic"** with a sub-section for time periods

These are all the same thing under different labels. You're looking for the section that lets you configure **time windows** with a **charge or discharge action**.

If you can't find it: the menu may be under "Advanced" or require an installer password. [Whirlpool 9xv6wp84](https://forums.whirlpool.net.au/thread/9xv6wp84) and [9n111qlk](https://forums.whirlpool.net.au/thread/9n111qlk) are the best places to ask if you're stuck.

## Step 3 - Switch the working mode to Economic Mode

Before you can configure a schedule, the inverter has to be in **Economic Mode** (the GoodWe term for "schedule-based operation"). Some app versions require you to explicitly select Economic Mode as the active working mode; others let you configure schedules from any mode and apply them when you switch.

Set the active working mode to **Economic Mode**. If the app warns you about overwriting other settings, that's expected - Economic Mode is the only mode that respects TOU schedules.

## Step 4 - Create the schedule slot

Add a new schedule entry with these settings:

| Setting | Value |
|---|---|
| **Start time** | `11:00` |
| **End time** | `14:00` |
| **Action / Mode** | **Charge** (not discharge, not export) |
| **SOC target** | `100%` (charge to full) |
| **Power source** | **Grid** (high priority) - or "Grid + PV" if your version separates them |
| **Days** | All days (Mon-Sun) |
| **Enabled** | Yes |

Save the schedule.

> **Why these specific settings?**
> - **11:00-14:00** matches the GloBird Zero Hero free window. If you're on a different free-window plan (some VPP variants are 10-13 or noon-3), adjust to match.
> - **Charge to 100%** because LFP chemistry tolerates regular full charges and the BMS calibrates itself at the top end. Charging to less than 100% leaves money on the table during peak.
> - **Grid priority** so the inverter prefers grid power for the charge (which is free during this window) and uses solar as a top-up. The opposite priority would consume your solar first and only top up from grid if solar wasn't enough - fine, but you'd export less.

## Step 5 - Verify it took effect

The next free window (11:00 the following day, or right now if you're inside one), check the inverter's behaviour:

- Battery SOC should rise toward 100% throughout the window.
- Grid import should be substantial (5-13kW depending on your inverter and battery state).
- Once SOC hits 100%, charging stops and the inverter holds - it does **not** drop into self-consumption and start discharging.

If the schedule didn't fire - battery isn't charging during the window - check:

- The inverter's working mode is actually set to Economic Mode (Step 3).
- The schedule is enabled.
- The inverter clock is correct. Run the [`goodwe_time_sync.yaml`](../automations/goodwe_time_sync.yaml) automation manually from HA, or check the time on the inverter's local web interface, to make sure it hasn't drifted.

## Critical: do not run Methods 1 or 2 alongside this

We say this in the strategy README and we'll say it again here. Method 1 (and any operation-mode change in Method 2) **deletes this TOU schedule** ([Whirlpool 9n111qlk](https://forums.whirlpool.net.au/thread/9n111qlk)). The first time HA writes "Eco mode" or "General mode" to `select.goodwe_inverter_operation_mode`, your carefully-set TOU slot is gone.

If you're running Method 3, your YAML in HA should never touch the operation mode - and the [Method 3 automation in this repo](../automations/globird/method3_hybrid/globird_zero_hero.yaml) is designed exactly that way. Don't add additional automations that change `select.goodwe_inverter_operation_mode` while you're on Method 3.

## What you should have at the end of this guide

- SolarGo set to Economic Mode.
- A daily 11:00-14:00 charge schedule, target 100% SOC, grid priority.
- Verified the schedule fires (next free window, watch battery SOC climb).
- No HA automations touching operation mode.

Next: [First-run checklist](./09_first_run_checklist.md).
