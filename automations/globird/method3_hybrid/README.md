# Method 3: Hybrid General Mode (Recommended)

> Before you copy anything: read the [strategy guide](../) to understand why this method exists and to create the required helpers. For terminology - including why "Eco Mode" in HA is called "Economic Mode" in the app, and what "TOU" actually means - see the [Glossary](../../../GLOSSARY.md).

This is the recommended method. The GoodWe app handles the free-window charge via a native TOU schedule; HA handles the smart layer at peak time (live SOC check, dynamic export limit, profit reporting). Neither system fights the other. HA never touches the operation mode, so it never overwrites flash and never deletes the SEMS+ schedule.

## Why this beats Method 1 - the throughput point

GoodWe ESA inverters can charge the battery at up to **13.5kW** when the firmware orchestrates 10kW of grid AC + 3.5kW of solar DC simultaneously. Home Assistant can only command the AC side via Eco Mode or the fast-charging switch, so a HA-driven charge tops out at the inverter's AC ceiling - about **10kW**.

That sounds small until you do the maths against a three-hour free window: 10kW x 3h = 30kWh vs 13.5kW x 3h = 40.5kWh. About a third more energy banked for free, every day. Over a year, that's nontrivial.

The mechanism: a SolarGo/SEMS+ TOU schedule tells the inverter firmware "charge to 100% between 11:00 and 14:00, grid priority". The firmware then blends grid AC + solar DC internally and the battery sees the combined input. HA can't replicate this because the API surface for "use both inputs at once" simply isn't exposed. References: [Whirlpool 9kppp8k2 - GoodWe ESA maximum charge rate](https://forums.whirlpool.net.au/thread/9kppp8k2), [mletenay #362](https://github.com/mletenay/home-assistant-goodwe-inverter/discussions/362).

There's also a behavioural reason: HA's `fast_charging_switch` exits forced-charge mode the moment SOC hits the target, after which the inverter reverts to self-consumption and starts draining the battery during the rest of the free window. The native TOU schedule charges to 100% and *holds* - no drain.

(And as a softer third reason: nothing here writes to the inverter's flash, so it's gentler on the EEPROM than Method 1. The flash-wear risk is community folklore - no documented bricked units - but there's no upside to writing flash when you don't have to.)

## How it works in practice

1. In the GoodWe SolarGo/SEMS+ app, you create a single **Economic Mode / TOU schedule** for the free window: 11:00-14:00, charge target 100% SOC, grid priority. The firmware does the rest.
2. In HA, this automation does one thing at 17:56: checks your SOC. If above the guard threshold, it arms the export limit to 5kW and records the baseline export reading. If too low, it drops the limit to 0W.
3. At peak end (21:01 by default, or 20:01 if you're on an older GloBird plan), HA restores the export limit to unrestricted and fires a profit notification.

The GoodWe inverter is in General Mode (self-consumption) the entire time from HA's perspective. HA never touches the operation mode - it only touches the export limit number entity.

## What you need

- Working native GoodWe integration (HACS not required).
- The entity `number.goodwe_grid_export_limit` (may be `_2` suffixed - check your entities) and `sensor.goodwe_battery_state_of_charge`.
- The seven helpers listed in the [strategy guide](../#required-home-assistant-helpers) - six shared plus `input_number.zero_hero_min_export_soc`.
- A notification device set up in HA.
- **A GoodWe app Economic Mode / TOU schedule** for the free window - see setup instructions below.

## GoodWe app setup (do this first)

In **SolarGo** or **SEMS+**, navigate to your inverter's settings and find the Economic Mode / TOU schedule section. Create a schedule slot:

- **Time:** 11:00 - 14:00
- **Mode:** Charge (not discharge)
- **SOC target:** 100%
- **Power source:** Grid (high priority)

This is the only thing you need to configure in the app. Do not add a separate discharge schedule - the inverter's self-consumption mode will naturally discharge the battery during the peak window because demand exceeds solar by that time of day, and the HA export limit controls exactly how much goes to the grid.

> **Note:** The app may call this "Economic Mode", "TOU Mode", or "Charge Schedule" depending on your app version and inverter model. It is the same thing. See the [Glossary](../../../GLOSSARY.md) for the full terminology breakdown.

## Install

1. Set up the GoodWe app schedule first (above).
2. Open [`globird_zero_hero.yaml`](./globird_zero_hero.yaml) and copy the whole file.
3. Go to **Settings > Automations & Scenes**, create a new empty automation, click the three dots > **Edit in YAML**, paste.
4. Replace every `# EDIT:` marker with your real entity IDs and notify service name.
5. Save, enable, and watch the first full day.

## Watch out for

- **SOC guard threshold.** Pulled from `input_number.zero_hero_min_export_soc` - set it to whatever SOC you need to keep in reserve for evening household load. 65% is a reasonable starting point for a 13.5kWh battery; tune by watching what SOC you actually end the night on.
- **Peak end time.** The YAML defaults to 21:01 (newer GloBird plans). If your plan ends at 20:00, change the `peak_end` trigger time to 20:01.
- **Super export cap.** Set the `input_number.zero_hero_super_cap` helper to `10` (older GloBird plan) or `15` (newer plan). The profit notification reads from this helper - do not leave it at the default without checking your plan.
- **Export limit entity name.** This is most likely `number.goodwe_grid_export_limit_2` (with `_2` suffix) on systems where the integration has been re-added. Check Developer Tools > States and use whichever has a live value.
- **App schedule conflicts.** Only have *one* Economic Mode schedule (the 11-14 charge). Do not also add an evening discharge schedule in the app - leave that to self-consumption.
- **Don't run Methods 1 or 2 alongside this.** Both Method 1 and any operation-mode change in Method 2 will delete your SEMS+ TOU schedule and silently break the free-window charge.
