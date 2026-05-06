# GloBird Zero Hero - Strategy Guide & Automations

This folder contains three different automations for managing a GoodWe ESA hybrid inverter on the **GloBird Zero Hero** plan. Each method takes a different approach to the same problem, with different tradeoffs.

Start here before copying any YAML. The wrong method for your setup will either break your SEMS+ schedules or, in the worst case, miss the peak window entirely and quietly cost you money.

---

## The plan, in two sentences

Zero Hero has a **free window** (11:00-14:00) where importing costs nothing, and a **peak window** (18:00-20:00 or 18:00-21:00 depending on when you signed up) where exporting pays you a premium rate on the first 10-15 kWh, plus a flat daily credit if you import nothing during the window.

The goal of the automations is to maximise what you earn in the peak window without cooking your battery or missing the free charge.

---

## How the windows work

### Free window (11:00 AM - 2:00 PM)

Power from the grid is free. The automation force-charges the battery to 100% from grid power if solar isn't keeping up.

**On charging LFP to 100% - it's fine.** Better than fine, actually. GoodWe ESA batteries are LFP (Lithium Iron Phosphate) chemistry, which doesn't share the "charging to 100% kills the battery" curse of older Li-ion packs. LFP cells *want* to hit 100% periodically - it's how the BMS calibrates itself. What you want to avoid is *sitting* at 100% for days on end; a daily top-up and then discharge is ideal. So charging hard during the free window and discharging during peak is exactly the cycle the chemistry wants.

### Peak window (6:00 PM - 8:00 or 9:00 PM)

Grid import is expensive. GloBird offers a high "Super Export" rate for the first 10 kWh you push back during this window (check your plan - some newer ones cap at 15 kWh), and a daily "Zero-Grid" credit (around $1.00) if you don't import anything during the window.

**The sweet spot is hitting exactly the Super Export cap**, not dumping the entire battery. You can keep exporting past the cap at the baseline rate (around 6c/kWh), but the wear-and-tear on the battery isn't worth the extra pennies. Discharge 10 kWh at 15c, bank the $1 credit, and knock off for the evening - about $2.50 for doing nothing.

---

## A word on what HA can and can't tell the inverter to do

To pick a method, you need to know two things about how the inverter actually responds when HA sends commands.

**1. HA can only command the AC side.** The GoodWe ESA can charge the battery at up to 13.5kW when the firmware blends 10kW grid AC + 3.5kW solar DC simultaneously, but the HA-facing API only exposes AC-side controls (Eco Mode, fast-charging switch, EMS power limit). A HA-driven charge tops out around 10kW. A SEMS+ TOU schedule, by contrast, lets the firmware orchestrate both inputs - and you get the full 13.5kW. Over a 3-hour free window, that's the difference between ~30 kWh and ~40 kWh banked. References: [Whirlpool 9kppp8k2](https://forums.whirlpool.net.au/thread/9kppp8k2), [mletenay #362](https://github.com/mletenay/home-assistant-goodwe-inverter/discussions/362).

**2. Changing operation mode from HA deletes any TOU schedule you set in the app.** When HA writes "Eco mode" or "General mode" to `select.goodwe_inverter_operation_mode`, it overwrites the TOU schedule slot stored in the inverter's flash ([Whirlpool 9n111qlk](https://forums.whirlpool.net.au/thread/9n111qlk)). This is why a lot of people try a "simple HA automation" and report their SEMS+ schedule mysteriously broke. There's also a flash-wear concern with frequent operation-mode writes - that's community folklore rather than a documented failure, but there's no upside to writing flash when you don't have to.

These two facts shape the three methods below. Method 1 accepts both consequences for the sake of simplicity. Method 2 dodges the schedule-deletion issue by using EMS RAM registers (a separate entity, never touches operation mode), but is still capped at the 10kW AC ceiling because HA still can't ask for DC blending. Only Method 3 dodges both - by leaving operation mode alone *and* delegating the charge to a SEMS+ TOU schedule that the inverter firmware can run at the full 13.5kW.

---

## The three methods

### Baseline: SEMS+ only (no HA automation)

Not recommended, but worth mentioning. You can skip the automations entirely, set a charge/discharge schedule in the GoodWe SEMS+ app, and just use HA to monitor. Simple. Safe.

**Downsides:** SEMS+ controls total inverter power, not grid export specifically - so it's imprecise. And it's a static schedule: it won't react to your battery being low, or household load spiking, or anything else. If your battery is at 25% going into peak, SEMS+ will happily try to discharge it anyway and buy grid power at peak rates to fill the gap. Not ideal.

### [Method 1: Standard Eco Mode](./method1_standard/)

The straightforward approach. Switch the inverter to Eco Mode at 11:00 AM (to force charge), back to General at 14:00, back to Eco at 18:00 (to force discharge), and back to General at the end of peak.

- Pro: Uses the native GoodWe integration. No HACS required.
- Pro: Straightforward logic - one automation, one entity to control.
- Con: Charges at ~10kW max (no DC blending - see "what HA can tell the inverter" above).
- Con: **Overwrites your SEMS+ TOU schedule** every time it changes mode.

**Pick this if:** you don't use SEMS+ for scheduling, you're happy for HA to own the timing entirely, and the ~30% throughput hit during the free window is acceptable to you.

### [Method 2: EMS RAM Commands](./method2_ems/) - experimental

Use the community-maintained GoodWe Experimental integration (HACS) to send Energy Management System commands directly to the inverter's RAM. Sets mode to `Charge` at 11:00, back to `Auto` at 14:00, to `Discharge` (or `Export AC` depending on firmware) at 18:00, back to `Auto` at the end of peak.

- Pro: Never touches operation mode - preserves any SEMS+ TOU schedule you've set.
- Pro: `Discharge` / `Export AC` mode covers house load first, then exports exactly the target Watts.
- Pro: Watchdog automation included (returns inverter to Auto if HA crashed mid-export).
- Con: Requires the experimental HACS integration ([mletenay/home-assistant-goodwe-inverter](https://github.com/mletenay/home-assistant-goodwe-inverter)).
- Con: EMS option strings vary by firmware - needs verification before it works.
- Con: A future GoodWe firmware update could change EMS registers and silently break it.
- Con: Still subject to the 10kW AC charging ceiling (same limitation as Method 1).

**Pick this if:** you're comfortable with HACS, you want a setup where your SEMS+ schedule survives, and you accept that the integration could break without notice.

### [Method 3: Hybrid General Mode (recommended)](./method3_hybrid/)

The free charging window is handled natively by a GoodWe **Eco Mode / TOU** schedule set directly in the SolarGo/SEMS+ app (11:00-14:00, target 100% SOC, grid priority). The firmware blends grid AC and solar DC to charge at up to 13.5kW and holds at 100% once the target is hit - no HA intervention in the free window.

HA handles the smart layer: at 17:56 it evaluates SOC, arms or blocks the 5kW peak export limit, and sends a nightly profit notification.

> **Terminology note:** "Eco Mode" in the SolarGo/SEMS+ app = what the GoodWe HA integration calls "Eco Mode" in `select.goodwe_inverter_operation_mode`. Despite sharing a name, the HA integration's `fast_charging_switch` is a *different* mechanism - it force-charges via a dedicated register, not via the TOU schedule slot, and only runs the AC side. See [GLOSSARY.md](../../GLOSSARY.md) for the full breakdown.

- Pro: Charges at the full 13.5kW (10 AC + 3.5 DC blended) during the free window.
- Pro: Native firmware handles the "charge then hold" behaviour correctly.
- Pro: Never touches operation mode from HA - TOU schedule is safe.
- Pro: Uses only the native HA integration - no HACS required.
- Pro: Dynamically protects against low-SOC export fails.
- Con: Requires one schedule in the GoodWe app + one HA automation instead of just one of either.

**Pick this if:** you want the thing that works well and keeps working. This is the default recommendation.

---

## Which method should I pick?

```
Default answer: Method 3. The 13.5kW vs 10kW charging difference alone
                makes it worth the slightly fiddlier two-system setup.

If you can't or won't use the SolarGo/SEMS+ app for scheduling:
├── Method 1 - straightforward, fully HA-driven, ~10kW charge ceiling.
└── Method 2 - preserves SEMS+ but needs HACS and depends on an
              experimental integration. Same 10kW ceiling. Pick this
              only if you specifically want HA to own the timing
              without overwriting any TOU schedule you've set.
```

## Don't mix methods

Pick one method per inverter. Method 1 changes operation mode, which **deletes the SEMS+ TOU schedule** that Method 3 relies on ([Whirlpool 9n111qlk](https://forums.whirlpool.net.au/thread/9n111qlk)). Method 2 itself is safe alongside Method 3 (it never touches operation mode), but if you mix-and-match while testing it's easy to lose your schedule by accident. Disable old automations before enabling a new method.

---

## Required Home Assistant helpers

All three methods use HA "helpers" for configuration and tracking. Before copying any YAML, create these in **Settings > Devices & services > Helpers**.

Six are shared across all methods. Methods 1, 2, and 3 each add one or two extras - see the per-method tables below. It's a bit annoying, but it means you can tune rates and toggle the whole thing on/off without touching the YAML, which you'll thank yourself for the first time GloBird adjusts prices.

### 1. Master enable/disable toggle
Gates the **peak window** behaviour - the SOC guard, the export limit changes, and the profit notification. Useful when away, or when you're running heavy loads and want to keep the battery full instead of exporting it.

What this toggle does **not** stop: in Method 1 the 11:00-14:00 force-charge fires regardless (the toggle only gates peak), and in Method 3 the free-window charge is owned by your SEMS+ TOU schedule and runs regardless of HA. To stop the free-window charge, you'd need to disable the automation itself (Method 1) or remove/disable the SEMS+ schedule (Method 3).

- **Helper type:** Toggle
- **Name:** `Zero Hero Enabled`
- **Icon (optional):** `mdi:flash`
- **Resulting entity ID:** `input_boolean.zero_hero_enabled`

### 2. Export baseline tracker
Stores your total-export-so-far at 18:00 each day, so the automation can calculate exactly how much you exported *during* peak (not including everything you'd already exported earlier in the day). The automation manages this automatically - don't edit it manually.

- **Helper type:** Number
- **Name:** `Zero Hero Export Start`
- **Minimum value:** `0`
- **Maximum value:** `100000`
- **Step size:** `0.1`
- **Resulting entity ID:** `input_number.zero_hero_export_start`

### 3. Super export rate ($/kWh)
The high rate GloBird pays for the first chunk of your peak export. This is the *total* rate you receive per kWh (which GloBird structures internally as base rate + bonus, but you just enter what lands in your account). As of writing this is $0.15/kWh - check your current plan.

- **Helper type:** Number
- **Name:** `Zero Hero Rate Super`
- **Minimum value:** `0`
- **Maximum value:** `2`
- **Step size:** `0.01`
- **Unit of measurement:** `AUD/kWh`
- **Initial value:** `0.15`
- **Resulting entity ID:** `input_number.zero_hero_rate_super`

### 4. Base export rate ($/kWh)
The rate GloBird pays for export beyond the Super Export cap. Usually around $0.06/kWh.

- **Helper type:** Number
- **Name:** `Zero Hero Rate Base`
- **Minimum value:** `0`
- **Maximum value:** `1`
- **Step size:** `0.01`
- **Unit of measurement:** `AUD/kWh`
- **Initial value:** `0.06`
- **Resulting entity ID:** `input_number.zero_hero_rate_base`

### 5. Daily zero-grid credit ($)
The flat credit GloBird pays if you don't import anything during the peak window. Usually around $1.00.

- **Helper type:** Number
- **Name:** `Zero Hero Rate Credit`
- **Minimum value:** `0`
- **Maximum value:** `10`
- **Step size:** `0.01`
- **Unit of measurement:** `AUD`
- **Initial value:** `1.00`
- **Resulting entity ID:** `input_number.zero_hero_rate_credit`

### 6. Super export cap (kWh)
The kWh threshold above which your export drops from the super rate to the base rate. Older GloBird Zero Hero plans cap at 10 kWh; some newer plans cap at 15 kWh. Check your plan documents.

- **Helper type:** Number
- **Name:** `Zero Hero Super Cap`
- **Minimum value:** `0`
- **Maximum value:** `30`
- **Step size:** `1`
- **Unit of measurement:** `kWh`
- **Initial value:** `10` (or `15` if you're on a newer plan)
- **Resulting entity ID:** `input_number.zero_hero_super_cap`

If GloBird changes any of these rates, you change the helper value in the HA UI. No YAML edits required.

### Method-specific extras

In addition to the six shared helpers above, create these depending on which method you picked:

**Method 1 (Standard Eco Mode):**

- `input_number.zero_hero_eco_power` - magnitude for Eco Mode Power. Read the **UNIT-TRAP** comment block at the top of the YAML before setting this; the value depends on whether your integration version uses percentages or watts. Min `0`, max `5000`, step `1`. Initial value: `100` (percentage) or `5000` (watts) once you've verified which.
- `input_number.zero_hero_min_export_soc` - SOC% floor below which peak export is blocked. Min `0`, max `100`, step `1`, unit `%`, initial value `65`.

**Method 2 (EMS RAM Commands):**

- `input_boolean.zero_hero_force_safe` - panic switch. Flip ON to force the watchdog to return the inverter to Auto on the next 5-minute tick. Default off.
- `input_number.zero_hero_min_export_soc` - same as Method 1 above.

**Method 3 (Hybrid):**

- `input_number.zero_hero_min_export_soc` - same as above. (Tune to suit your battery size and overnight load. Start at `65`. If you consistently wake up with battery to spare - meaning you didn't use as much overnight as the threshold protected for - drop it a few percent so peak export arms more aggressively. If you wake up at 0%, raise it.)

---

## Quick sanity check before you start

- Your GoodWe integration is installed and working (you can see `sensor.goodwe_battery_state_of_charge` or similar in HA).
- Your HA Companion App is set up for notifications - you'll want these firing during the first few days.
- If going with Method 2, the experimental HACS integration is installed and you can see `select.goodwe_ems_mode` (or similar).
- You've got the six helpers above.
- You've read the strategy, picked a method, and have the right folder open.

Right - you're ready. Go into the method folder you picked and follow the README there.

---

## What about closed-loop optimisation? (Predbat, EMHASS)

The three methods above use **fixed time windows** matched to the GloBird Zero Hero plan. If you outgrow that - say you switch to Amber Electric or any wholesale-spot plan where prices change every 5-30 minutes - you'll want a system that decides each day's charge/discharge schedule from forecast solar, household load history, and live prices, instead of hardcoded windows.

Two community projects already do this and both can drive a GoodWe ESA via the experimental integration:

- **[Predbat](https://github.com/springfall2008/batpred)** - runs a 48-hour optimisation every few minutes, originally built for GivEnergy but extended to other inverters including GoodWe. Reads Solcast forecasts and your tariff windows, decides when to charge and discharge, and drives the inverter through HA entities.
- **[EMHASS](https://github.com/davidusb-geek/emhass)** - linear-programming optimisation for PV/load/price scheduling, more configurable but a heavier setup.

Neither is maintained by this repo and neither has GloBird-specific recipes - for Zero Hero's fixed windows the gain is small, and Method 3 is simpler. But if you find yourself wanting the automation to react to *forecast* rather than *clock time*, that's where to go next.

