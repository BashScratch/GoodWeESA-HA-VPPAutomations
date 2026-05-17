# Method 4: Hybrid General Mode (Recommended)

> Before you copy anything: read the [strategy guide](../) to understand why this method exists and to create the required helpers. For terminology - including why "Eco Mode" in HA is called "Economic Mode" in the app, and what "TOU" means - see the [Glossary](../../../GLOSSARY.md).

This is the recommended method. The GoodWe app handles the free-window charge via a native TOU schedule; HA handles the smart layer at peak time (live SOC check, dynamic export limit, profit reporting). Neither system fights the other. HA never touches the operation mode, so it never overwrites flash and never deletes the SEMS+ schedule.

## Why this beats Method 2 - the throughput point

The GoodWe ESA 10kW model (GW9.999K-EHA-G20) can charge the battery at up to **13.5kW** when the firmware combines grid AC and solar DC simultaneously. This is the published spec - see GoodWe's official ESA Series datasheet ([V2.1 April 2026 PDF](https://admin.goodwe.com/Api/downloadFile?id=3448&mid=60&type=2)), "Max. Charging Power" row. Home Assistant can only command the AC side via Eco Mode or the fast-charging switch, so a HA-driven charge tops out at the inverter's nominal AC ceiling - about **10kW**.

That sounds small until you do the maths against a three-hour free window: 10kW for 3 hours equals 30kWh, vs 13.5kW for 3 hours equals 40.5kWh. About a third more energy banked for free, every day. Over a year, that's nontrivial.

**Caveat on when this matters in practice:** if you've got a large battery (around 48kWh and up) where 30kWh in 3 hours doesn't fill it, or you're charging an EV during the free window, this throughput gap is real money. The architecture detail that makes the EV case work: the battery prefers DC (solar), so AC capacity can be redirected to the EV charger while the battery still gets full charge from PV. On a smaller battery (say 13.5kWh) with no EV, you'll be at 100% well before 14:00 either way and the gap is largely academic. **The 13.5kW figure is also model-specific** - the 5kW ESA tops out at 7.5kW battery charging, the 8kW ESA at 12kW. See the [model compatibility table](../#model-and-firmware-compatibility) in the strategy guide.

**If you're on a three-phase ESA (the middle letter of the suffix is T - e.g. GW9.999K-ETA-G20 instead of GW9.999K-EHA-G20), the throughput advantage doesn't apply to you.** Three-phase ESAs cap max charging at nominal AC across the whole range, with no AC+DC blending headroom. Method 4 is still worth using on three-phase for the other reasons (precise grid-export control, your TOU schedule survives, all the HA smart-layer benefits), just not for the charge-rate boost. See the [strategy guide's three-phase table](../#three-phase-esa---charge-rates-by-model) for the per-model numbers.

The mechanism: a SEMS+ TOU schedule tells the inverter firmware "charge to 100% between 11:00 and 14:00, grid priority". The firmware then orchestrates grid AC + solar DC together and the battery sees the combined input. HA can't replicate this because the API surface for "use both inputs at once" simply isn't exposed. Community confirmation of the same effect: Whirlpool thread "Goodwe ESA maximum charge rate?" - explanation by user **nutttr** with confirmation from **Zerosignal** ([thread link](https://forums.whirlpool.net.au/thread/9kppp8k2)).

There's also a behavioural reason: HA's `fast_charging_switch` exits forced-charge mode the moment SOC hits the target, after which the inverter reverts to self-consumption for the rest of the free window. In practice that meant the battery would start discharging into whatever the house was running, including an EV charger if one was plugged in - exactly the opposite of what you want during the free window, where the grid power is free and the EV should be pulling from the grid, not the battery. The native TOU schedule charges to 100% and **holds** - the battery doesn't discharge to anything (house, EV, or otherwise) for the rest of the free window. That's the behaviour that matters for households charging an EV during 11:00-14:00.

## Honest note about flash writes

We previously claimed Method 4 doesn't touch the inverter's flash memory. **That's not accurate.** Writing to `number.goodwe_grid_export_limit` over Modbus appears to write to the inverter's persistent storage (flash/EEPROM) on several GoodWe ESA models. Method 4 does this twice a day (peak start at 17:56 and peak end at 21:01), so it has flash exposure too - just less than Method 2's four daily operation-mode writes.

How worried should you be? Probably not very, but worth knowing:

- Typical flash write-cycle ratings are 100,000+ cycles. Two writes a day for 10 years is ~7,300 cycles - well inside the rated lifetime.
- No bricked-inverter cases from this pattern have surfaced in the community that we're aware of.
- The chip type (true EEPROM with limited writes vs flash with much higher endurance) isn't publicly documented.

If you specifically want **zero flash writes**, use **[Method 3 (EMS RAM Commands)](../method3_ems/)** instead. EMS commands target volatile RAM and never touch flash. The tradeoff is the experimental HACS dependency and the mandatory watchdog (covered in Method 3's README).

For most users on a single-phase 10kW ESA who care about charging throughput, Method 4 is still the right pick. The flash exposure is real but moderate, and the 30% throughput advantage during the free window is the bigger lever.

## How it works in practice

1. In the GoodWe SEMS+ app, you create two TOU schedules: a charge slot (11:00-14:00, target 100% SOC) for the free window, and a discharge slot (peak window, 100% inverter power) for HA to manage on top. The firmware runs both.
2. In HA, this automation does one thing at 17:56: checks your SOC. If above the guard threshold, it arms the export limit to 5kW and records the baseline export reading. If too low, it drops the limit to 0W.
3. At peak end (21:01 by default, or 20:01 if you're on an older GloBird plan), HA restores the export limit to unrestricted and fires a profit notification.

The GoodWe inverter is in General Mode (self-consumption) the entire time from HA's perspective. HA never touches the operation mode - it only touches the export limit number entity.

## What you need

- Working native GoodWe integration. **HACS is optional but recommended.** The core of Method 4 runs on entities the native integration exposes (`number.goodwe_grid_export_limit`, `sensor.goodwe_battery_state_of_charge`). The midnight-reset has one belt-and-braces line that turns off `switch.goodwe_fast_charging_switch` as a safety net; that switch only exists with the experimental HACS integration ([mletenay/home-assistant-goodwe-inverter](https://github.com/mletenay/home-assistant-goodwe-inverter)). On a native-only install the action errors silently (the YAML uses `continue_on_error: true`) and the rest of the automation continues. Method 4 *will* run native-only, you just lose that one safety line. Install HACS too if you want it working.
- The entity `number.goodwe_grid_export_limit` (may be `_2` suffixed if you've installed and removed the integration before - check your entities) and `sensor.goodwe_battery_state_of_charge`.
- The nine helpers listed in the [strategy guide](../README.md#required-home-assistant-helpers) - six shared plus three Method 4 extras: `input_number.zero_hero_min_export_soc` (SOC floor for arming peak export), `input_number.zero_hero_peak_export` (target peak grid-export limit in Watts, default 5000), and `input_number.zero_hero_max_export` (your inverter's nominal AC ceiling that HA restores at peak end).
- A notification device set up in HA.
- **Two SEMS+ TOU schedules**: a charge slot for the free window, plus a discharge slot for the peak window (HA manages grid export within it). See setup instructions below and the dedicated [TOU setup prerequisite guide](../../../prerequisites/08_sems_tou_schedule.md).

## GoodWe app setup (do this first)

In **SEMS+** (recommended) or SolarGo, navigate to your inverter's settings and find the **TOU** section. Create two schedule slots: a **Charge** slot for the free window and a **Discharge** slot for the peak window.

> **Why SEMS+ over SolarGo?** GoodWe is migrating consumer features into SEMS+ over time, and TOU controls have the most polish there. If your SEMS+ install doesn't show TOU yet (older app version), update it; if it still doesn't appear, fall back to SolarGo. Be aware that the two apps don't share installer-level passwords - if SEMS+ asks for one and your SolarGo password gets rejected, that's expected. See [prereq 08](../../../prerequisites/08_sems_tou_schedule.md) for the detail.

**Charge slot (free window):**

- **Time:** 11:00 to 14:00
- **Mode:** Charge
- **SOC target:** 100%
- **Power source:** Grid (high priority)

**Discharge slot (peak window):**

- **Time:** 18:00 to 21:00 (or 18:00 to 20:00 if you're on the older GloBird plan)
- **Mode:** Discharge
- **Power:** 100% of inverter output
- **SOC target:** whatever you want as your discharge floor (a low number like 10% lets HA's SOC guard be the actual floor)

Why both slots? The discharge slot at 100% gives the inverter headroom to push 5kW to the grid *plus* whatever the house is drawing, simultaneously. Without it, the inverter is in self-consumption during peak - which works, but caps the inverter's output at house demand. With the discharge slot at 100%, HA's `number.goodwe_grid_export_limit` is the actual control: it caps grid export at 5kW, and the inverter covers house load on top of that.

**Critical concept: "discharge power" in SEMS+ TOU means *total inverter output*, not grid-export specifically.** A 10% setting on a 10kW inverter means 1kW total (house + grid combined), not 1kW to the grid. This is why we set discharge power to 100% in SEMS+ and use HA's grid-export-limit as the precise lever.

The full step-by-step including SEMS+ menu navigation lives at [prerequisites/08_sems_tou_schedule.md](../../../prerequisites/08_sems_tou_schedule.md). The summary above is what to enter in the app.

## Install

1. Set up the GoodWe app schedule first (above).
2. Open [`globird_zero_hero.yaml`](./globird_zero_hero.yaml) and copy the whole file.
3. Go to **Settings > Automations & Scenes**, create a new empty automation, click the three dots > **Edit in YAML**, paste.
4. Replace every `# EDIT:` marker with your real entity IDs and notify service name.
5. Save, enable, and watch the first full day.

## Watch out for

- **SOC guard threshold.** Pulled from `input_number.zero_hero_min_export_soc` - set it to whatever SOC you need to keep in reserve for evening household load. 65% is a reasonable starting point for a 13.5kWh battery; tune by watching what SOC you end the night on.
- **Peak end time.** The YAML defaults to 21:01 (newer GloBird plans). If your plan ends at 20:00, change the `peak_end` trigger time to 20:01.
- **Super export cap.** Set the `input_number.zero_hero_super_cap` helper to `10` (older GloBird plan) or `15` (newer plan). The profit notification reads from this helper - do not leave it at the default without checking your plan.
- **Export limit entity name.** This is most likely `number.goodwe_grid_export_limit_2` (with `_2` suffix) on systems with both the native and experimental integrations installed. Check Developer Tools > States and use whichever has a live value (a non-zero number that updates).
- **Export limit master switch.** Some GoodWe ESA installs also expose `switch.goodwe_grid_export_limit_switch` which is the master enable for the export-limit feature. If yours has this and it's off, writes to `number.goodwe_grid_export_limit` will silently do nothing. Flip the switch on once and leave it on.
- **App schedule layout.** Two TOU slots is correct for Method 4: a charge slot (11:00-14:00) and a discharge slot (18:00-21:00 at 100% inverter output). Don't add other slots in the app - they'll fight HA's logic at peak.
- **Late-afternoon solar export blocked when SOC guard fails.** When the 17:56 SOC guard sees not-enough-charge for peak, it sets the export limit to 0W. This protects the battery for overnight as designed, but it ALSO blocks any solar surplus your system would otherwise have exported between 17:56 and 18:00 (or later in winter when solar is still generating during peak). The 0W stays in place until peak end (21:01), when HA restores the limit. On a sunny day where you've already filled the battery, that's fine - the 0W limit doesn't matter. On the borderline-SOC days where the guard kicks in, you lose a small amount of late-afternoon solar export. Accepted tradeoff: the alternative would be more complex SOC-aware logic that decides "block grid discharge but allow solar passthrough", and the simple 0W limit is fine for the typical case.
- **Don't run Method 2 or 3 alongside this.** Method 2 directly overwrites your SEMS+ TOU schedule on every mode change. Method 3 itself doesn't touch operation mode, but if you experiment with it and accidentally trip an Eco/General mode write, the TOU schedule is gone. Pick one method per inverter and stick with it.
