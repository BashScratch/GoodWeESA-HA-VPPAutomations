# Method 1: App-only (no Home Assistant)

> Before you copy anything: read the [strategy guide](../) to understand why Methods 2-4 add HA on top of this one. If you're unsure what "TOU" or "Economic Mode" means, see the [Glossary](../../../GLOSSARY.md).

> **Heads up: the GoodWe app is a moving target.** SEMS+ and SolarGo are evolving quickly, with new fields and menu reorganisations appearing in most releases. We try to keep these instructions current, but a screen we describe might already have an extra field, a renamed label, or a different layout by the time you open it. If a step here doesn't match what you see in the app, it's almost always a recent app update rather than a fundamental change to the inverter - check the GoodWe ESA Facebook group or the Whirlpool thread for the current state, then come back and the rest of the guide will still apply.
>
> **Specific moving pieces worth knowing about (as of 18 May 2026):** recent SEMS+ releases (paired with recent battery firmware) have started showing two new fields in the TOU **Time Period** dialog. Both are **visible in the UI but not editable / not yet functional** - GoodWe is rolling them out and the consensus from community reports is that the fields appear as a preview but don't enforce anything yet. Wide rollout (with the fields actually working) is expected within the next month. Until then, the older app-only paths below remain the right way to go.
>
> 1. **Export Power Limit (per TOU period).** A toggle plus a watts value that, once functional, will pin the grid-export rate for that specific TOU window. When it lands, this is the field that will replace the installer-menu **Soft Power Limit** workaround (the "Andrew Palmer approach", covered later in this README) - same outcome, no installer password, set inside the TOU dialog where it belongs. **Not editable today**, so keep using the Soft Power Limit setup for now. We'll update this guide when the field becomes usable.
> 2. **Discharge SOC limit (per TOU discharge slot).** A per-window floor that, once functional, would sit inside the TOU dialog itself - separate from the system-wide Battery Protection menu (see Step 5 below) which is the battery's last-resort floor where the inverter stops discharging and starts importing from the grid in any operating mode. The new TOU field is a different concept: it would let you say "stop the TOU discharge at 30% but still let the battery cover overnight household load down to the Battery Protection floor below that". Currently visible in the dialog on some installs but not editable; community reports are mixed about when (and on which installs) it'll start enforcing. Watch for it but don't rely on it yet.
>
> Battery Protection remains your universal floor regardless of how the new TOU fields behave when they go live.

The simplest possible Zero Hero setup. Everything happens in the GoodWe SEMS+ app (or SolarGo if that's what you've already got working). No Home Assistant, no HACS, no helpers, no YAML. The inverter's firmware does the work; you check the results in the app.

This is the right starting point if any of the following apply:

- You haven't set up Home Assistant yet and want a working baseline before adding it.
- You'd rather not run extra software just for energy automation.
- You want to understand what the inverter does on its own before deciding whether HA is worth the effort.
- You want a reference point to compare the HA methods against.

If you already know you want HA, skip this and go to [Method 4 (Hybrid)](../method4_hybrid/) - the recommendation. But come back here if you want a clean explanation of how TOU works at the inverter level, since the HA methods all build on top of it.

## What this method does

Two TOU schedule slots in SEMS+:

- **Charge slot:** 11:00-14:00. The inverter charges the battery toward your Charge Cut-off SOC target from the grid (free during the Zero Hero window) and from any solar that's available.
- **Discharge slot:** 18:00-21:00 (or 18:00-20:00 on older Zero Hero plans). The inverter discharges the battery into your house and out to the grid, capped at the discharge power you set.

That's the whole automation. Outside those windows the inverter sits in self-consumption (use solar first, fill from battery, import from grid only if needed).

## What you need

- A GoodWe ESA inverter wired up and online.
- The SEMS+ app on your phone, logged in to your GoodWe account. SolarGo also works (you'll need it specifically for the Soft Power Limit setup further down, since that menu hasn't moved to SEMS+ yet); we recommend SEMS+ for the TOU work since GoodWe is migrating consumer features there.
- A GloBird Zero Hero plan (or similar dynamic plan with fixed free + peak windows).
- That's it. No Home Assistant, no integrations, no helpers, no YAML.

## Setup

The full step-by-step including SEMS+ menu navigation lives at [prerequisites/08_sems_tou_schedule.md](../../../prerequisites/08_sems_tou_schedule.md). The summary:

### Step 1 - Switch the inverter to TOU / Economic mode

In SEMS+, navigate to your inverter's settings and set the working mode to **TOU** (called "Economic Mode" on older app versions - same thing). This is the only mode that respects time-window schedules.

### Step 2 - Create the charge slot

In the SEMS+ TOU dialog, pick the **Charge** tab. The fields you'll see:

| Field | Value |
|---|---|
| **Start Time** | `11:00` |
| **End Time** | `14:00` |
| **Repeat** | All months (Jan-Dec selected), all days (Mon-Sun selected) |
| **Charge Cut-off SOC** | `100%` for maximum Super Export headroom at peak; or `90%` if you want to be deliberately conservative on cycle wear (see the warranty note in the [strategy guide](../#how-the-windows-work) - the GoodWe residential battery warranty doesn't specify an SOC ceiling, but a lower cut-off reduces daily throughput a touch). This is the *target* the inverter charges toward; charging stops once SOC hits this. |
| **Grid Import Charging Power** | `100%` (charge as fast as the inverter will let it) |

Note: there's no separate "Power Source: Grid" field as such. Setting Grid Import Charging Power to 100% tells the inverter to charge from the grid at full rate; solar contributes on top via the DC bus during the same window (this is the firmware-managed AC+DC blending that gets you up to 13.5kW on the single-phase 10kW model).

Save the slot.

### Step 3 - Create the discharge slot

Switch to the **Discharge** tab in the same TOU dialog. As of writing the discharge tab only exposes one knob you can actually change - **Discharge Power** as a percentage of inverter capacity. There's no SOC floor in the TOU dialog itself (the SOC floor is set separately in the **Battery Protection** menu in SEMS+ - see Step 4 below). The fields:

| Field | Value |
|---|---|
| **Start Time** | `18:00` |
| **End Time** | `21:00` (or `20:00` on older Zero Hero plans) |
| **Repeat** | All months, all days |
| **Discharge Power** | Percentage of inverter capacity. See "Discharge power, the gotcha" below for how to pick a value. |

If your version of SEMS+ shows extra fields in this dialog (Export Power Limit, Discharge SOC limit, Rated Current of the Incoming Circuit Breaker), see the callout at the top of this README. Those fields are appearing in the UI on recent app/firmware combinations but aren't editable yet; treat them as previews of features coming in the next month or so. For now, set Discharge Power as in the table above and use the Soft Power Limit section further down for precise grid export.

### Step 4 - Set the SOC floor in Battery Protection

In SEMS+, separately from the TOU dialog, find the **Battery Protection** (or similar) menu. This is where you set the SOC at which the inverter stops discharging and starts pulling from the grid to cover house load. A reasonable starting point is `20-30%` - leaves some headroom for overnight household use after the peak window.

When the battery hits this floor during peak discharge, the inverter switches to importing from the grid to cover any remaining house load. That import will cost you the Zero-Grid daily credit if it happens during the peak window, so set the floor high enough that you don't run out before peak ends.

### Step 5 - Verify it works

Watch the next free window:

- Battery SOC climbs toward 100% (or 90% if that's your target) between 11:00 and 14:00.
- Once SOC hits the target, charging stops.

Watch the next peak window:

- Battery SOC drops as the inverter discharges into the house and grid.
- Discharge stops when SOC hits the Battery Protection floor.

That's it. The inverter handles the rest day after day.

## The discharge-power gotcha (read this twice)

The TOU **Discharge Power** field is a **percentage of inverter capacity**, and that percentage represents **total inverter output** - house load and grid export combined - not grid export specifically.

So on a 10kW inverter, setting Discharge Power to 50% means "discharge at up to 5kW total". Of that 5kW, the house gets first call and whatever's left goes to the grid.

Worked example - working out what % to set if you want to export 5kW to the grid on a 10kW inverter:

1. Estimate your typical house load during peak hours (e.g. 2kW between 6pm and 9pm if you're cooking and have lights on).
2. Add your target grid export (e.g. 5kW for Zero Hero with the 15kWh super cap over 3 hours).
3. Total output you want: 2kW house + 5kW grid = 7kW.
4. As a percentage of the 10kW inverter: 70%. Set Discharge Power to 70%.

It's not perfect. House load varies - if the house is drawing 5kW (dryer, oven, kettle all on) you're at the inverter's 7kW ceiling and grid export drops to 2kW. If the house is drawing 0.5kW, grid export climbs to 6.5kW. The total inverter output is what you've pinned, not grid export specifically.

**The better way to do this in Method 1** is to leave Discharge Power at 100% and use the **Soft Power Limit** setting (in the SolarGo installer menu) to pin grid export specifically. That setup is covered in the next section. With it, the inverter discharges at house-load + your chosen grid-export number regardless of house load fluctuation. Worth the installer-password hassle if you care about hitting the Zero Hero credits precisely. (Once GoodWe enables the in-TOU **Export Power Limit** field for editing - it's already visible but not functional, see the callout at the top - this section becomes obsolete; we'll update the guide when that lands.)

## Going further: precise grid export with the Soft Power Limit (Andrew Palmer's approach)

Method 1 *can* pin grid export to a specific number without HA. The trick is the **Soft Power Limit** setting in SolarGo's installer-level menu, combined with TOU discharge at 100%.

Naming note: GoodWe's own docs sometimes call this **"Software Power Limit"** (see [GoodWe's Export Power Limit Solution page](https://en.goodwe.com/export-power-limit-solution)), while community discussion mostly says "Soft Power Limit". Same thing. You'll find it in SolarGo at **Advanced Setting > Power Limit > Software Power Limit** (installer password required - see prereq 01 for that).

This approach was documented by **Andrew Palmer** on the GoodWe ESA Facebook group ([original post](https://www.facebook.com/groups/1639058747083230/posts/1725207231801714/)). Crediting him because the configuration combination isn't obvious from GoodWe's own docs.

### How it works

- **Soft Power Limit** caps the inverter's grid-export rate (different from total inverter output - this one's specifically about what goes to the grid).
- TOU discharge at 100% gives the inverter full headroom to discharge.
- The inverter then outputs `(Soft Power Limit) + (current house load)` total. Grid export sits at the Soft Power Limit, house load is covered on top.

Worked example with Soft Power Limit = 2kW:

- House load 1kW, battery output = 3kW (1kW house + 2kW grid). Grid export 2kW.
- House load 5kW, battery output = 7kW (5kW house + 2kW grid). Grid export still 2kW.
- House load drops to 0.5kW, battery output = 2.5kW. Grid export still 2kW.

That's the precise grid export the discharge-power gotcha above said wasn't achievable in app-only mode. The Soft Power Limit makes it achievable.

### Setup (requires installer password)

1. Open SolarGo (this setting may not be in SEMS+ yet; if it is, look for the same path in SEMS+ first).
2. Log in with the **installer password**. The default is `1234` unless your installer set a custom one. SEMS+ uses a different installer password to SolarGo (see [prereq 01](../../../prerequisites/01_enable_modbus_on_inverter.md) for the gotcha).
3. Navigate to **Settings > Advanced Settings > Power Limit**.
4. Set **Soft Power Limit** to your desired grid-export rate. For Zero Hero with the 10kWh super cap, **2000W** over 3 hours gives you 6kWh exported - well under the 10kWh super cap, with comfortable margin. Tune higher if you have a larger battery and want to hit the cap exactly; lower if you want a smaller daily target.
5. Save and exit.

### What this gives you, and what it doesn't

Method 1 with the Soft Power Limit gets you:

- Precise grid export at your chosen rate.
- House load fully covered from the battery (no grid imports during peak, so the daily Zero-Grid credit is preserved).
- All without HA.

What you still don't get without HA:

- **Pre-peak SOC guard.** If the battery is short on charge going into peak, the inverter still discharges down to the Battery Protection floor and then starts importing from the grid at peak rates. Method 4 blocks discharge to the grid entirely on low-SOC days, preserving what charge is left for the house. Method 1 can't decide "today's not the day, skip peak export".
- **Notifications and profit reporting.** No nightly summary of what you exported and earned.
- **Helper-tunable rates.** If GloBird changes the super rate, you don't have a one-tap UI for adjusting calculations.

If those gaps don't bother you, Method 1 with Soft Power Limit is a clean setup. If they do, Method 4 layers them on top.

### Modbus alternative (skip the installer password)

If you've enabled Modbus TCP on the inverter (see [prereq 01](../../../prerequisites/01_enable_modbus_on_inverter.md)), you can adjust Soft Power Limit via Home Assistant's `number.goodwe_grid_export_limit` entity without needing the installer password. This is what Method 4's automation does dynamically. If you're going to use HA anyway, you might as well use Method 4 - the Soft Power Limit approach is for users who want to avoid HA entirely.

## What you're missing without HA

This method gets you the basic charge-during-free / discharge-during-peak cycle, which is most of the value. But there are real things HA adds that the app alone can't:

- **Precise grid-export control during peak.** SEMS+ pins total inverter output; HA's `number.goodwe_grid_export_limit` pins grid export specifically. With Zero Hero's "first 15kWh at 15c, rest at 6c" structure, hitting the cap precisely matters. The Soft Power Limit workaround above closes this gap from inside the app at the cost of an installer-password trip. (When GoodWe finishes the in-TOU Export Power Limit rollout - see the callout at the top - this gap will close natively in the consumer app and the Soft Power Limit detour won't be needed.)
- **Dynamic SOC guard.** SEMS+ will discharge down to the Battery Protection floor regardless of conditions. If the battery's at 40% going into peak because the free-window charge didn't fill it (cloudy day, late free-window start, etc.), it'll discharge to the floor and then buy grid power at peak rates to cover the rest. HA can decide "today's not the day, skip peak export" based on live SOC at 17:56.
- **Profit notifications.** SEMS+ shows you what charged and discharged but doesn't compute "you exported X kWh tonight at the super rate, plus the daily credit, total $Y." HA does that and pushes it to your phone each evening.
- **Helper-tunable rates.** When GloBird adjusts the super rate or daily credit (it happens), updating SEMS+ tariff configuration is fiddly. With HA the rates live in number helpers you can edit from the dashboard in two clicks.

If those things matter to you, jump to:

- [Method 2 (Standard Eco Mode)](../method2_standard/) - HA-driven, simpler than Method 4 but with tradeoffs.
- [Method 3 (EMS RAM Commands)](../method3_ems/) - the right tool if you're on a dynamic-pricing VPP like Amber.
- [Method 4 (Hybrid - recommended)](../method4_hybrid/) - keeps this method's TOU schedule and adds HA on top for the smart layer. The default recommendation for fixed-window plans like Zero Hero.

If they don't, this method is fine. You're getting most of the Zero Hero benefit with none of the complexity.

## Watch out for

- **The discharge-power semantics.** Read the gotcha above twice. It's the most common confusion.
- **Inverter clock drift.** GoodWe inverters can drift a few minutes per week. If your clock drifts and your TOU charge slot fires from 11:04 to 14:04 instead of 11:00 to 14:00, you've lost ~7% of the free window. Check the inverter clock against your phone's clock periodically and resync via the app if needed. Methods 2-4 include an HA automation that does this daily.
- **Firmware availability for the 13.5kW combined-charging capability.** On the single-phase 10kW ESA, the firmware that combines grid AC and solar DC for 13.5kW battery charging has been rolling out from around April 2026. Some firmware releases have it, some don't, and some users have had to ask GoodWe Level 2 support for a standalone push. If you're seeing your battery cap at ~10kW during the free window despite plenty of solar, this is the likely cause.
- **Plan rate changes.** GloBird occasionally adjusts the super rate, base rate, or daily credit. Check your latest bill and your plan documents periodically.
- **No safety-net SOC guard.** If your battery is going into peak with limited charge (cloudy day, free window cut short, etc.), this method will discharge down to the Battery Protection floor (set in the Battery Protection menu of SEMS+, not the TOU dialog) and then start importing from the grid at peak rates to cover any remaining demand. There's no "skip peak today" logic. HA methods add that.
- **Soft Power Limit isn't dynamic.** If you decide to change your peak export rate (e.g. you want 1.5kW instead of 2kW for a few days), it's a manual trip into the SolarGo installer menu each time. With Method 4, the export limit lives in a HA helper you can edit from the dashboard in two clicks (or via automation if you want it to vary by day of week, weather forecast, etc.). Most users set the Soft Power Limit and forget; if you want to tune it often, that's a real tradeoff to consider. (Once the in-TOU Export Power Limit field becomes editable, the same dynamic-tuning gap will apply to it - one TOU edit per change.)

## Method 1 with Soft Power Limit vs Method 4 - they're closer than you'd think

Once you've set up the Soft Power Limit, Method 1 is functionally close to Method 4 in what it achieves at the inverter level. Both pin grid export precisely, both let the inverter charge at full 13.5kW via TOU. The differences are the things HA brings on top:

- Method 4 adds the SOC guard logic that decides whether to arm peak export each day.
- Method 4 adds notifications, profit calc, and helper-tunable rates.
- Method 4 lets you change the export limit dynamically rather than via the installer-menu Soft Power Limit.

If those things matter, Method 4 is worth the HA setup. If they don't, Method 1 with Soft Power Limit gets you almost the same energy outcome with a much simpler maintenance footprint.

(When the in-TOU Export Power Limit field becomes editable - see the callout at the top - the same comparison will apply but without the installer-menu detour.)

## Going further

If this method works for you, brilliant - you're done. If you want any of the things in the "What you're missing" list above, [Method 4 (Hybrid)](../method4_hybrid/) is the natural next step. It keeps the TOU schedules you've just set up and adds HA's smart layer at peak time.
