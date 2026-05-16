# GloBird Zero Hero - Strategy Guide & Automations

This folder contains four methods for managing a GoodWe ESA hybrid inverter on the **GloBird Zero Hero** plan. Method 1 is app-only (no Home Assistant required). Methods 2, 3, and 4 add HA on top in different ways.

Start here before copying any YAML. The wrong method for your setup will either break your SEMS+ schedules or, in the worst case, miss the peak window entirely and quietly cost you money.

---

## The plan, in two sentences

Zero Hero has a **free window** (11:00-14:00) where importing costs nothing, and a **peak window** (18:00-20:00 or 18:00-21:00 depending on when you signed up) where exporting pays you a premium rate on the first 10-15 kWh, plus a flat daily credit if you import nothing during the window.

The goal of the automations is to maximise what you earn in the peak window without cooking your battery or missing the free charge.

---

## How the windows work

### Free window (11:00 AM - 2:00 PM)

Power from the grid is free. The automation force-charges the battery to 100% from grid power if solar isn't keeping up.

**Charging LFP to 100% - generally fine.** GoodWe ESA batteries use LFP (Lithium Iron Phosphate) chemistry, which is well known to tolerate high state-of-charge significantly better than older NMC or Li-ion packs. LFP also benefits from periodic full charges because its flat discharge curve makes voltage-based SOC estimates drift over time; a full charge resets the BMS coulomb counter and keeps the SOC reading honest. (If you want to read deeper on the cycle-life difference between LFP and NMC chemistries, [this Sandia / J. Electrochem. Soc. study](https://iopscience.iop.org/article/10.1149/1945-7111/abae37) is one of the more cited references - though their specific test conditions don't perfectly mirror the Zero Hero usage pattern, so treat it as background reading rather than a direct number we're claiming applies here.)

What the warranty actually says (and doesn't): we audited the current GoodWe Australia/NZ residential battery warranty document for the BAT-D series (Rev 1.0, July 2025, covers GW5.1-BAT-D and GW8.3-BAT-D - these are the Lynx D modules most ESA installs stack to make their usable capacity) and the commercial-ESA warranty document (Rev 1.2, October 2025, covers the GW125/261-ESA-LCN-G10 industrial cabinet). **Neither document specifies a recommended SOC operating range** - there is no "stay between 10% and 90%" rule. Earlier drafts of this guide stated one; we've corrected that. What the residential warranty does specify:

- **10 years to 70% of Usable Energy retained, OR** the Minimum Through Output Energy (3 MWh per kWh of usable capacity), whichever comes first.
- *"This Warranty covers a capacity equivalent to one full cycle per day."* So the warranty is designed around households that cycle the battery roughly once a day. The document doesn't say more than one cycle a day voids coverage; it says coverage is sized at that rate.
- The battery's "Usable Energy" sits inside a slightly larger physical-cell capacity (e.g. 8 kWh usable vs 8.32 kWh rated on the GW8.3-BAT-D-G20), so the "100%" you see in SEMS+ or HA is approximately 96-98% of the cell range. The buffer is small (a few percent), not huge.
- Cell voltage range 2.85V-3.6V, conservative on both ends versus LFP's broader 2.5V-3.65V envelope.

How does Zero Hero compare to that 1-cycle-per-day design rate? It depends on two things: your battery size, and how much of your daily household load comes off the battery (which depends on solar coverage, household size, and what else is plugged in). Daily battery discharge is bounded by whichever is smaller: your battery's usable kWh, or your household's daily discharge demand. The table below brackets two realistic cases on a newer GloBird plan (15 kWh Super Export cap):

- **Lighter user case (~15 kWh/day total discharge):** roughly the Super Export cap plus minimal household battery use. Solar covers most daytime load, the free window covers a chunk of evening prep, modest overnight base load. Battery is mostly working the Super Export.
- **Heavier user case (~30 kWh/day total discharge):** Super Export plus ~15 kWh of household load the battery actually covers (the evening + overnight + early-morning windows where solar and the free window aren't running). A typical Australian household averaging around 800W draw uses ~19 kWh/day total; the battery covers ~15 kWh of that once solar and free-grid contributions are subtracted.

| Usable | Typical stack | Daily discharge: lighter user (~15 kWh demand) | Daily discharge: heavier user (~30 kWh demand) | Years to throughput cap (light to heavy) |
|---|---|---|---|---|
| 5 kWh | 1 × GW5.1-BAT-D | 5 (battery-cap) | 5 (battery-cap) | 8.2 to 8.2 |
| 8 kWh | 1 × GW8.3-BAT-D | 8 (battery-cap) | 8 (battery-cap) | 8.2 to 8.2 |
| 16 kWh | 2 × GW8.3-BAT-D | 15 | 16 (battery-cap) | 8.8 to 8.2 |
| 24 kWh | 3 × GW8.3-BAT-D | 15 | 22 (near-cap) | 13.2 to 9.0 |
| 32 kWh | 4 × GW8.3-BAT-D | 15 | 30 | 17.5 to 8.8 |
| 40 kWh | 5 × GW8.3-BAT-D | 15 | 30 | 21.9 to 11.0 |
| 48 kWh | 6 × GW8.3-BAT-D | 15 | 30 | 26.3 to 13.2 |

(GW5.1-BAT-D stacks of 1-6 modules give 5, 10, 15, 20, 25, 30 kWh usable. Same math applies - the 5.1 modules are also rated at 3 MWh/kWh throughput.)

Read: **small batteries (≤16 kWh) on Zero Hero cycle at or near the warranty's design rate of 1/day in both user cases**, and the throughput cap hits around year 8 just shy of the 10-year time trigger. **Larger batteries (32 kWh+) sit well below the design cycle rate** in the lighter-user case, but a heavier user with full Super Export plus meaningful household battery dependency can still push them down toward year 9-13. To estimate your own runway: pull your average daily battery discharge from HA over a couple of months, then divide your battery's lifetime throughput budget (3 MWh × your usable kWh) by your annual discharge.

If you'd rather stay deliberately conservative on any battery size, set your TOU charge target to 90% instead of 100%. The LFP wear difference between 90% and 100% top-of-charge is small but the daily throughput drops a touch, which extends your runway. You'll miss a little Super Export headroom in the peak window in exchange.

### Peak window (6:00 PM - 8:00 or 9:00 PM)

Grid import is expensive. GloBird offers a high "Super Export" rate for the first 10 kWh you push back during this window (check your plan - some newer ones cap at 15 kWh), and a daily "Zero-Grid" credit (around $1.00) if you don't import anything during the window.

**The sweet spot is hitting exactly the Super Export cap**, not dumping the entire battery. You can keep exporting past the cap at the baseline rate (around 6c/kWh), but the wear-and-tear on the battery isn't worth the extra doubloons. Discharge 10 kWh at 15c, bank the $1 credit, and knock off for the evening - about $2.50 for doing nothing.

---

## A word on what HA can and can't tell the inverter to do

To pick a method, you need to know two things about how the inverter responds when HA sends commands.

**1. HA can only command the AC side.** The GoodWe ESA's 10kW model (GW9.999K-EHA-G20) can charge the battery at up to **13.5kW** when the firmware combines grid AC and solar DC simultaneously. This is published spec - see GoodWe's official ESA Series datasheet, "Max. Charging Power" row ([GoodWe ESA Series datasheet PDF, V2.1 April 2026](https://admin.goodwe.com/Api/downloadFile?id=3448&mid=60&type=2)). The inverter's nominal AC power is 9.999kW; the extra ~3.5kW comes from solar DC bypassing the AC stage and going directly to the battery via the MPPTs. The HA-facing API only exposes AC-side controls (Eco Mode, fast-charging switch, EMS power limit), so a HA-driven charge tops out around 10kW. A SEMS+ TOU schedule, by contrast, lets the firmware orchestrate both inputs and gives you the full 13.5kW. Community confirmation of the same effect: Whirlpool thread "Goodwe ESA maximum charge rate?", explanation by user **nutttr** with confirmation from **Zerosignal** ([thread link](https://forums.whirlpool.net.au/thread/9kppp8k2)); also discussed at [mletenay #362](https://github.com/mletenay/home-assistant-goodwe-inverter/discussions/362).

The throughput gap (10kW vs 13.5kW) matters most if you have a large battery (around 48kWh and up) where 30kWh in 3 hours doesn't fill it, or if you're charging an EV during the free window. With concurrent EV charging the architecture detail matters: the battery prefers DC (solar), so AC capacity can be redirected to the EV while the battery still gets full charge from PV. On a smaller battery (say 13.5kWh) with no EV, you'll be at 100% well before 14:00 either way, so the gap is largely academic.

**2. Changing operation mode from HA deletes any TOU schedule you set in the app.** When HA writes "Eco mode" or "General mode" to `select.goodwe_inverter_operation_mode`, it overwrites the TOU schedule slot stored in the inverter's persistent storage ([Whirlpool thread "GoodWe ESA - Setting export TOU with SOC limit"](https://forums.whirlpool.net.au/thread/9n111qlk)). This is why a lot of people try a "simple HA automation" and report their SEMS+ schedule mysteriously broke.

There's also a hypothetical wear concern with frequent operation-mode writes. Whether the inverter stores these in EEPROM or flash (community discussion uses both terms - the underlying chip type isn't documented publicly), the typical write-cycle rating is 100,000+ cycles. At the rate HA fires mode changes (a handful per day at most), that's likely more than the inverter's service life. No bricked-inverter cases have surfaced in the community. Treat it as "no upside to writing persistent storage when you don't have to" rather than a concrete risk.

These facts shape Methods 2-4 below. **Method 1** sidesteps both by not using HA at all - the inverter's TOU schedule does everything, no HA mode writes, no HACS. **Method 2** accepts both consequences for the sake of HA simplicity. **Method 3** dodges the schedule-deletion issue by using EMS RAM registers (a separate entity, never touches operation mode), but is still capped at the 10kW AC ceiling because HA still can't ask for DC blending. Only **Method 4** dodges all of it, by leaving operation mode alone *and* delegating the charge to a SEMS+ TOU schedule that the inverter firmware can run at the full 13.5kW.

---

## Model and firmware compatibility

GoodWe publishes two ESA datasheets - one for single-phase, one for three-phase. The two product lines have different architecture, and the throughput rationale for Method 3 lands very differently on each.

### Single-phase ESA - charge rates by model

From the official [ESA Series single-phase datasheet, V2.1 April 2026](https://admin.goodwe.com/Api/downloadFile?id=3448&mid=60&type=2):

| ESA model | Nominal AC | Max battery charging power |
|---|---|---|
| GW3K-EHA-G20 (3kW) | 3.0kW | 4.5kW |
| GW3.6K-EHA-G20 (3.6kW) | 3.6kW | 5.4kW |
| GW5K-EHA-G20 (5kW) | 5.0kW | 7.5kW |
| GW6K-EHA-G20 (6kW) | 6.0kW | 9.0kW |
| GW8K-EHA-G20 (8kW) | 8.0kW | 12.0kW |
| **GW9.999K-EHA-G20 (10kW)** | **9.999kW** | **13.5kW** |

Note the 1.35x multiplier between nominal AC and max charging power. That gap is the AC+DC blending headroom: the firmware can charge from grid (limited by AC) AND from PV (over the DC bus) simultaneously to reach the higher number. **This is the throughput advantage Method 3 unlocks.** HA-driven charging only commands the AC side and tops out at the nominal AC figure.

Firmware on older inverters may need pushing. A few community reports describe the 13.5kW capability not being live on inverters with older firmware. The April 2026 V2.1 datasheet is now the published spec, but if you're on older firmware and don't see combined grid+solar charging during a TOU charge slot, contact GoodWe support and ask for the latest firmware to be pushed. Their Level 2 support has been responsive on this.

### Three-phase ESA - charge rates by model

From the official [ESA Series three-phase datasheet, V2.1 April 2026](https://admin.goodwe.com/Api/downloadFile?id=4072&mid=60&type=2):

| ESA model | Nominal AC | Max battery charging power |
|---|---|---|
| GW5K-ETA-G20 (5kW) | 5.0kW | 5.0kW |
| GW6K-ETA-G20 (6kW) | 6.0kW | 6.0kW |
| GW8K-ETA-G20 (8kW) | 8.0kW | 8.0kW |
| GW9.999K-ETA-G20 (10kW) | 9.999kW | 10.0kW |
| GW12K-ETA-G20 (12kW) | 12.0kW | 12.0kW |
| GW15K-ETA-G20 (15kW) | 15.0kW | 15.0kW |
| GW20K-ETA-G20 (20kW) | 20.0kW | 20.0kW |
| GW25K-ETA-G20 (25kW) | 25.0kW | 25.0kW |
| GW29.999K-ETA-G20 (30kW) | 29.999kW | 30.0kW |

**Three-phase ESAs don't have the AC+DC blending headroom.** Max charging equals nominal AC across the entire range. So if you're on a three-phase ESA, **the throughput advantage Method 4 has on single-phase doesn't apply to you** - HA-driven charging and TOU-scheduled charging both top out at the same number.

**The per-phase trap.** A "15kW three-phase inverter" is effectively three separate 5kW inverters in a trench coat - one per phase. The 15kW headline number is the sum, not what any single phase can deliver. So during the discharge window, if one of your phases is drawing more than 5kW (e.g. a high-load circuit on phase A while phases B and C are idle), the inverter cannot push the other phases' spare capacity across to make up the difference - it'll start pulling from the grid on the busy phase instead. On Zero Hero that means a brief peak-window grid import, which forfeits the daily $1 Zero-Grid credit.

If you're on a three-phase ESA and finding the credit gets blown despite your battery being well-charged, this is the likely cause. The fix is on the electrician side: get your distribution board rebalanced so peak-time household loads are spread evenly across phases. Aim for none of your phases to draw more than 5kW (or whatever per-phase rating your inverter has) during the peak window. Worth raising with your installer if it's a recurring problem.

Method 4 is still the recommended approach on three-phase, just for different reasons. You still get:

- Precise grid-export control via `number.goodwe_grid_export_limit` (Method 1's app-only baseline TOU and Methods 2/3 set total discharge, which is imprecise; Method 1 with Andrew Palmer's Soft Power Limit gets around this).
- Your SEMS+ TOU schedule isn't deleted by HA mode changes.
- The HA smart layer (SOC guard, profit notifications, helper-tunable rates) is unchanged.

Just don't expect a 35% throughput bump from switching from app-only to Method 4 on a three-phase setup.

### Naming tip

You can spot single-phase vs three-phase by the **middle letter** of the suffix: single-phase ESAs are `E**H**A-G20` (e.g. GW9.999K-EHA-G20), three-phase are `E**T**A-G20` (e.g. GW9.999K-ETA-G20). Same length, same `A-G20` ending - it's the H vs T that tells you which you're looking at. The H is for "Home" (single-phase residential) and the T is for "Three-phase".

### Modbus TCP

Off by default on every GoodWe ESA we've seen. You'll need to enable it before any of the HA integration will work. See [prerequisites/01_enable_modbus_on_inverter.md](../../prerequisites/01_enable_modbus_on_inverter.md).

---

## The four methods

### What HA brings to Methods 2-4

Method 1 is app-only. The other three methods all add HA on top, so before the per-method differences, here's what you get from putting HA in front of any of them versus running SEMS+ alone:

- **Pre-peak SOC guard.** HA checks battery SOC at 17:56 and either arms or blocks peak export entirely based on whether you have enough headroom. SEMS+ TOU can set an SOC floor for the discharge slot, but it can't decide "don't discharge at all today, save what's there for overnight load" - it'll discharge to the floor and then start buying grid power at peak rates to cover the rest of demand. HA's all-or-nothing arming logic avoids that.
- **Conditional notifications.** "Zero Hero Armed" / "Aborted" / "Complete" messages so you know what happened each day without having to check the app.
- **Profit calculation.** Nightly notification with kWh sold, super-vs-base split, daily credit, and total. Neither SEMS+ nor SolarGo computes this against your tariff structure.
- **Helper-tunable rates.** Tariff changes are a number edit in the HA UI, not a schedule rebuild in the app.
- **Master enable/disable toggle.** One switch pauses peak behaviour without deleting anything (handy when away or when you want to keep the battery full for a known heavy load).

Methods 2-4 inherit this. The differences below are about how each method talks to the inverter, not about whether you get the smart layer.

### [Method 1: App-only (no HA)](./method1_app_only/)

The simplest possible setup. Two TOU schedule slots in SEMS+ (charge during free window, discharge during peak), no Home Assistant.

- Pro: Zero software dependencies. Works as soon as the inverter is online.
- Pro: The inverter's firmware does the orchestration. Charge can hit the full 13.5kW (single-phase 10kW model, AC+DC blended) since it's a native TOU schedule.
- Pro: Nothing to break, nothing to maintain.
- Con: Discharge "power" is total inverter output, not grid-export specifically. Your grid export drifts with house load.
- Con: No dynamic SOC guard. If your battery is short on charge going into peak, the inverter still discharges to its SOC floor and you start buying grid power at peak rates to cover the rest.
- Con: No notifications, no profit reporting.
- Con: Static schedule - doesn't react to anything.

**Pick this if:** you want the simplest possible Zero Hero setup, you don't have HA (or don't want it for this), or you want to learn how TOU works at the inverter level before adding HA on top.

### [Method 2: Standard Eco Mode (HA-driven)](./method2_standard/)

The straightforward approach. Switch the inverter to Eco Mode at 11:00 AM (to force charge), back to General at 14:00, back to Eco at 18:00 (to force discharge), and back to General at the end of peak.

- Pro: Straightforward logic - one automation, one entity to control.
- Pro: Inherits the HA smart-layer benefits above (SOC guard, notifications, profit calc, tunable rates).
- Con: **Requires the experimental HACS integration** because `number.goodwe_eco_mode_power` is only exposed there.
- Con: Charges at ~10kW max (no DC blending - see "what HA can tell the inverter" above).
- Con: **Overwrites your SEMS+ TOU schedule** every time it changes mode.
- Con: The 5kW peak target is *total inverter discharge*, not grid-export specifically - same imprecision as the app-only baseline. If house load drops mid-peak, more goes to the grid; if house load spikes, less does (or nothing does, if house load exceeds the target).

**Pick this if:** you don't use SEMS+ or SolarGo for scheduling, you're happy for HA to own the timing entirely, you're comfortable with HACS, and the ~30% throughput hit during the free window is acceptable. Honest take: this method is dated compared to the other three. It's a carry-over from how earlier-generation GoodWe batteries used to be controlled, before TOU schedules and the EMS RAM register got useful. Still workable, but Methods 1, 3, and 4 are all better choices today.

### [Method 3: EMS RAM Commands](./method3_ems/) - experimental

Use the community-maintained GoodWe Experimental integration (HACS) to send Energy Management System commands directly to the inverter's RAM. Sets mode to `Charge` at 11:00, back to `Auto` at 14:00, to `Discharge` (or `Export AC` depending on firmware) at 18:00, back to `Auto` at the end of peak.

- Pro: Never touches operation mode - preserves any SEMS+ TOU schedule you've set.
- Pro: `Discharge` / `Export AC` mode covers house load first, then exports exactly the target Watts.
- Pro: Watchdog automation included (returns inverter to Auto if HA crashed mid-export).
- Pro: Inherits the HA smart-layer benefits (SOC guard, notifications, profit calc, tunable rates).
- Pro: **Comes into its own on dynamic wholesale plans** (Amber Electric, OVO Charge Anytime forecast tiers, etc.) where you want HA to drive mode changes every few minutes based on live price signals. The EMS register is the right tool for that pattern. For a fixed-window plan like Zero Hero, this advantage doesn't land.
- Con: Requires the experimental HACS integration ([mletenay/home-assistant-goodwe-inverter](https://github.com/mletenay/home-assistant-goodwe-inverter)).
- Con: EMS option strings vary by firmware - needs verification before it works.
- Con: A future GoodWe firmware update could change EMS registers and silently break it.
- Con: Still subject to the 10kW AC charging ceiling (same limitation as Method 2).
- Con (uncertain): If you've also got a SEMS+ TOU schedule active, the precedence between TOU and EMS during a real conflict isn't fully documented. Working hypothesis: EMS RAM commands win while in effect (until inverter restart or until the EMS state is cleared). If confirmed, this is fine. If not, the two could fight at boundaries. Worth verifying on your install before relying on it.

**Pick this if:** you're on a dynamic-pricing VPP that needs frequent mode changes, OR you want a setup where your TOU schedule survives in the app and you accept that the integration could break without notice.

### [Method 4: Hybrid General Mode (recommended)](./method4_hybrid/)

The free charging window is handled natively by a GoodWe **TOU** schedule set directly in the SEMS+ app (11:00-14:00, target 100% SOC, grid priority). The firmware blends grid AC and solar DC to charge at up to 13.5kW and holds at 100% once the target is hit - no HA intervention in the free window.

HA handles the smart layer: at 17:56 it evaluates SOC, arms or blocks the 5kW peak export limit, and sends a nightly profit notification.

> **Terminology note:** What the SEMS+ app calls "TOU" or "Economic Mode" is the same thing the HA integration exposes as "Eco mode" in `select.goodwe_inverter_operation_mode`. Despite sharing the "Eco" name, the HA integration's `fast_charging_switch` is a *different* mechanism - it force-charges via a dedicated register, not via the TOU schedule slot, and only runs the AC side. See [GLOSSARY.md](../../GLOSSARY.md) for the full breakdown.

- Pro: Charges at the full 13.5kW (10 AC + 3.5 DC blended) during the free window, where the inverter and battery support it.
- Pro: Native firmware handles the "charge then hold" behaviour correctly.
- Pro: Never touches operation mode from HA - your TOU schedule is safe.
- Pro: Inherits the HA smart-layer benefits (SOC guard, dynamic export limit, notifications, profit calc, tunable rates).
- Pro: `number.goodwe_grid_export_limit` is precise grid-export control (not "total discharge" like Methods 1, 2, and 3). If house load varies, your grid-export number stays the same (provided the inverter has the headroom to cover both house and grid simultaneously).
- Pro: **Mostly insulated from firmware changes.** Uses the native HA integration's documented entities plus the GoodWe app's own TOU feature, both of which GoodWe maintains. Less exposed to breakage than Method 3's experimental-register approach.
- Caveat: Method 4 still writes to the inverter's persistent storage twice daily via `number.goodwe_grid_export_limit`. Less flash exposure than Method 2 (4 writes/day), but more than Method 3 (zero, since EMS targets RAM). If you specifically want zero flash writes, Method 3 is the right pick. For most users the throughput advantage of Method 4 outweighs this; see Method 4's README for the write-cycle math.
- Pro/Con: One small experimental-only dependency. The midnight reset turns off `switch.goodwe_fast_charging_switch` as a safety net (legacy from when an earlier version of this automation used the fast-charge switch as the primary charging mechanism; now that the charge is owned by TOU, this line just catches the case where someone manually flipped the switch on and forgot). That entity only exists with the HACS integration; on a native-only install the action errors silently and the rest of the automation continues (the YAML uses `continue_on_error: true`). So Method 4 *will* run native-only, you just lose one belt-and-braces line of safety.
- Con: Requires two TOU schedules (charge + discharge) in the SEMS+ app, plus one HA automation. More moving parts than the other methods.

**Pick this if:** you want the thing that works well and keeps working. This is the default recommendation.

---

## Which method should I pick?

Three recommendations cover most cases. Method 2 exists for completeness but isn't recommended over the other three (it has the heaviest flash-write footprint and Method 4 dominates it on every other axis).

- **[Method 1: App-only](./method1_app_only/)** if you don't want to run Home Assistant at all. With Andrew Palmer's Soft Power Limit setup it gets you precise grid export and Zero-Grid credit preservation without any HA. The simplest path; nothing to maintain beyond two TOU slots in SEMS+. A solid choice if HA isn't already in your life.

- **[Method 4: Hybrid](./method4_hybrid/)** for most people who want the happy middle ground. The things HA is good at (SOC guard, notifications, profit calc, tunable rates) layered on top of the inverter doing the heavy lifting via TOU schedules. Charges at the full 13.5kW (single-phase 10kW model) thanks to firmware-managed AC+DC blending. The default recommendation for most Zero Hero users.

- **[Method 3: EMS RAM Commands](./method3_ems/)** if you want to fully lean into HA-driven inverter control - either because you enjoy the tinkering or because you're planning to move to a dynamic-pricing plan like Amber Electric that needs frequent mode changes. EMS RAM commands are the right tool for fast price-following, and zero flash writes is a bonus. Worth picking even on a fixed-window plan like Zero Hero if you want to learn HA-side inverter control deeply before switching plans.

If none of those quite fit, the per-method READMEs above have the detailed pros/cons.

## Don't mix methods

Pick one method per inverter. Method 2 changes operation mode, which **deletes the SEMS+ TOU schedule** that Methods 1 and 4 rely on ([Whirlpool 9n111qlk](https://forums.whirlpool.net.au/thread/9n111qlk)). Method 3 itself is safe alongside Methods 1 and 4 (it never touches operation mode), but if you mix-and-match while testing it's easy to lose your schedule by accident. Disable old automations before enabling a new method.

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

Method 1 (app-only) doesn't need any HA helpers - the inverter's TOU schedule does the work.

Methods 2-4 use the six shared helpers above plus a few extras depending on which one you pick:

**Method 2 (Standard Eco Mode):**

- `input_number.zero_hero_eco_power` - magnitude for Eco Mode Power. Read the **UNIT-TRAP** comment block at the top of the YAML before setting this; the value depends on whether your integration version uses percentages or watts. Min `0`, max `5000`, step `1`. Initial value: `100` (percentage) or `5000` (watts) once you've verified which.
- `input_number.zero_hero_min_export_soc` - SOC% floor below which peak export is blocked. Min `0`, max `100`, step `1`, unit `%`, initial value `65`.

**Method 3 (EMS RAM Commands):**

- `input_boolean.zero_hero_force_safe` - panic switch. Flip ON to force the watchdog to return the inverter to Auto on the next 5-minute tick. Default off.
- `input_number.zero_hero_min_export_soc` - same as Method 2 above.

**Method 4 (Hybrid):**

- `input_number.zero_hero_min_export_soc` - same as above. (Tune to suit your battery size and overnight load. Start at `65`. If you consistently wake up with battery to spare - meaning you didn't use as much overnight as the threshold protected for - drop it a few percent so peak export arms more aggressively. If you wake up at 0%, raise it.)
- `input_number.zero_hero_max_export` - inverter's nominal AC export ceiling in Watts. **Set this to your specific inverter's nameplate maximum.** Method 4 restores the export limit to this value at peak end (21:01). Examples: `10000` for the 10kW ESA, `8000` for the 8kW, `5000` for the 5kW. Sending `10000` to a smaller inverter's export limit could throw an out-of-bounds error. Min `0`, max `30000`, step `100`, unit `W`. Initial value: whatever your inverter is rated for.

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

