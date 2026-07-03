# Glossary - GoodWe / GloBird / Home Assistant Terminology

One of the most confusing things about setting up a GoodWe ESA with Home Assistant on a GloBird plan is that the same concept is called something different in every interface: the GoodWe hardware docs, the SolarGo/SEMS+ app, the native HA integration, the experimental HACS integration, and the GloBird plan documentation all have their own vocabulary. This glossary maps them together.

> **Verification note:** Terms in the HA integration columns are verified against the native HA integration source and the experimental integration's GitHub README. GoodWe app terminology is verified against GoodWe user manuals and community reports. Where exact option strings in the experimental integration could not be confirmed from primary sources, this is noted explicitly.

---

## Inverter operating modes

| What it does | GoodWe hardware docs | SolarGo / SEMS+ app | HA native integration | HA experimental integration (mletenay HACS) |
|---|---|---|---|---|
| Normal self-consumption - use solar first, battery fills gaps, export surplus | Self-use Mode | Self-consumption / Self-use | `select.goodwe_inverter_operation_mode` = `General mode` | Same |
| Schedule-based charge or discharge using time windows | Economic Mode | TOU (current) / Economic Mode (older app) | `select.goodwe_inverter_operation_mode` = `Eco mode` | Same |
| Hold battery fully charged, pass grid to loads during outage | Backup Mode | Backup Mode | `select.goodwe_inverter_operation_mode` = `Backup mode` | Same |
| Set Eco Mode charge/discharge magnitude | Eco Power | (configured per TOU slot) | **Not exposed** | `number.goodwe_eco_mode_power` |
| Force-charge battery right now, as fast as possible | Fast Charge / Forced Charge | Charge Now | **Not exposed** | `switch.goodwe_fast_charging_switch` + `number.goodwe_fast_charging_soc` |
| Allow export up to a specific wattage limit | Export Power Limit / Feed-in Limit | Power Limit / Feed-in Limit | `number.goodwe_grid_export_limit` | `number.goodwe_grid_export_limit` |
| EMS - direct battery charge/discharge via RAM register command | EMS (Energy Management System) | Not exposed in app | **Not exposed** | `select.goodwe_ems_mode` + `number.goodwe_ems_power_limit` |
| Synchronise inverter clock to HA | (button) | (manual) | `button.goodwe_synchronize_inverter_clock` | Same |

### Architecture: HACS is a layer over native, not a replacement

The mletenay HACS integration is built as an **addition** to the native one. When both are installed (the standard setup for Methods 2-4), the native integration handles the read sensors and basic mode controls, and the HACS integration adds the active-control entities listed above. They don't conflict.

Per mletenay's repo: if you ever uninstall HACS, the native integration takes over seamlessly and existing entity IDs, history, and statistics are preserved. Migration in either direction is supported.

### Implications for the four methods

- **Method 2** uses `number.goodwe_eco_mode_power` for the charge/discharge magnitude, which is experimental-only. Method 2 therefore requires HACS, despite earlier framing in this repo that said it didn't.
- **Method 4** mostly runs on the native integration (operation mode + grid export limit + standard sensors). The midnight reset uses `switch.goodwe_fast_charging_switch` as a safety net, which is experimental-only. The YAML wraps that one action in `continue_on_error: true`, so on a native-only install the action errors silently and the rest of the automation keeps running. Method 4 still works fully native; you just lose the one belt-and-braces line.
- **Method 3** is fully experimental-dependent (uses the EMS entities).

### The "Eco Mode" / "Economic Mode" / "TOU" naming tangle

This is the most common point of confusion, and it comes from GoodWe using the same concept under slightly different names in different contexts:

- **"Economic Mode"** is what GoodWe's hardware documentation and the SolarGo/SEMS+ app call the scheduled charge/discharge operating mode. This is the mode where you set time windows for when the battery should charge, discharge, or hold.

- **"TOU" (Time of Use)** is the *scheduling feature type* within Economic Mode - it refers to the time-window schedule itself, not the mode name. When a GoodWe manual or forum post says "set up a TOU schedule", they mean "create charge/discharge time windows inside Economic Mode".

- **"Eco Mode"** is what the Home Assistant native integration (and the experimental one) calls this same mode in `select.goodwe_inverter_operation_mode`. The option string is `Eco mode`.

So: **HA "Eco mode" = app "Economic Mode"** - and both use TOU-style time scheduling. They are the same thing under two names.

**Critical gotcha:** When HA writes an Eco Mode command, it writes directly into EEPROM slot 1, overwriting whatever Economic Mode schedule you previously set in the app. The app and HA are not aware of each other's schedule data - HA simply replaces it. This is why Method 4 avoids touching the operation mode from HA entirely.

### EMS modes (experimental integration only)

The experimental integration exposes `select.goodwe_ems_mode` with several options. The GoodWe integration README describes the following scenarios, though the **exact option strings as they appear in the HA select entity may vary by inverter model and firmware** - always verify against your actual entity in Developer Tools > States before building automations:

| Scenario | What it does |
|---|---|
| Auto (self-use) | Normal self-consumption - battery responds to household demand |
| Charge | Force battery to charge from PV (high priority) or grid (low priority) |
| Discharge / Export battery | Battery discharges; PV covers house load first, then battery exports surplus |
| Export grid (sell to grid) | Targets a specific export wattage; PV preferred, battery fills gap |
| Charge from grid | Force charge from grid (high priority); PV used if available |

The Method 3 automation in this guide uses `"Charge"`, `"Discharge"`, and `"Auto"`. **If these option strings don't match your entity, check Developer Tools > States > `select.goodwe_ems_mode` for the actual available options and adjust the YAML accordingly.** Common alternative names for the discharge-with-export scenario include `"Export AC"` and `"Export grid"`.

---

## GloBird Zero Hero tariff terminology

| What it means | GloBird's term | What this guide calls it |
|---|---|---|
| 11:00-14:00 period where grid electricity costs nothing | "ZEROCHARGE" period / "Off-peak Usage" in your bill | Free window |
| 18:00-21:00 period where export earns a premium (older grandfathered plans end at 20:00) | "Super Export Window" / "ZEROHERO Window" | Peak window |
| High export rate during peak (e.g. $0.10/kWh total on QLD ZEROHERO, Jul 2026) | "ZEROWASTEDSOLAR" / "Super Export" | Super rate |
| Export rate outside peak but within 4pm-11pm (e.g. $0.02/kWh on QLD, Jul 2026) | "Solar/GenerationFeedin(4pm-11pm)" | Base rate |
| Daily credit for not importing during the peak window | "ZEROHERO" credit | Daily credit |
| kWh threshold above which super rate stops | "Super Export Threshold" / "Super Export Cap" | Super cap (15 kWh current; 10 kWh on older grandfathered plans) |
| Import threshold to retain the Zero-Grid credit | "ZEROHERO Threshold" | 0.03 kWh/hour per current GloBird key conditions = ~0.09 kWh (90 Wh) total across the 3-hour peak window |
| GloBird's special-event high-rate program (not relevant to GoodWe owners) | "ZEROLIMIT" / "Critical Peak" | See note below |

### Note on the super export rate structure

GloBird structures the super rate as: **base rate (4pm-11pm feed-in) + Super Export top-up = total super rate**. For QLD ZEROHERO as of the 1 July 2026 rate review this is $0.02 base + $0.08 top-up = $0.10 total. For the purposes of automation, you only care about the *total rate you receive* during peak ($0.10 in this example) vs what you receive once you've exceeded the cap ($0.02 same-window, $0.00 after 11pm). The helpers in these automations store and use the total figures. **Rates vary by state and review date** (GloBird reviews on 1 Jan and 1 Jul - the July 2026 review cut the QLD feed-in rates noticeably, so don't assume last quarter's numbers still hold). Check your current welcome pack for what you're actually on.

### Note on ZEROLIMIT (Critical Peak) and GoodWe

GloBird's current ZEROHERO offer lists a "ZEROLIMIT" benefit that pays **$1.00/kWh for exports** during nominated Critical Peak windows (dispatch events GloBird calls at their discretion, anecdotally triggered when the AEMO wholesale spot price spikes toward the [Market Price Cap, currently $20,300/MWh = $20.30/kWh for the 2025-26 financial year per the AEMC determination](https://www.aemc.gov.au/news-centre/media-releases/aemc-updates-market-price-cap-2025-26), often during heatwaves or grid-stress events). The "eligible brand" list in the welcome pack - AlphaESS, Anker, eCactus, Neovolt, Redback, SAJ, Sigenergy, SolaX, Solis + Dyness, Sungrow - is what GloBird can **dispatch directly** via the manufacturer's cloud control API. GoodWe isn't on that list because GloBird doesn't currently have a control hook into GoodWe's cloud, so they can't auto-discharge your battery for you when a Critical Peak event fires.

What this does NOT necessarily mean is that you can't earn the credit. Anecdotally (community reports, unverified), GloBird's Critical Peak payments appear to be settled from meter data - i.e. if your meter recorded exports during the nominated window, you get paid the top-up regardless of who initiated the discharge. So a GoodWe owner who reacts to a Critical Peak alert (or to a wholesale-price spike) by manually pushing the battery to export *during* the event could in principle collect the $1/kWh. **This is unconfirmed and may depend on your meter's 5-minute interval data, plan terms in your state, and any audit logic GloBird applies.** Don't bank on it; treat it as a possible upside, not a stable revenue stream.

If you want to chase this opportunistically, the technique is: watch AEMO spot price for your region (NSW1, QLD1, VIC1, SA1) via a HA integration like AEMO or OpenNEM, and when it spikes - or when you receive a Critical Peak email/SMS from GloBird - trigger a manual full-rate export through whichever Method you're using. We've parked an automation concept for this (F11 in the project's working notes) but haven't built it because we'd want a few confirmed-payment data points first.

---

## Home Assistant helper and entity types

| Term used in YAML / docs | What it actually is |
|---|---|
| `input_boolean` | A toggle switch you create as a helper - on or off. Persists across restarts. |
| `input_number` | A number input you create as a helper - stores a value you can edit in the UI. Persists across restarts. |
| `number.*` | A number entity exposed directly by an integration (e.g. the inverter's export limit register). Different from `input_number` - you don't create these, the integration does. |
| `switch.*` | A binary on/off entity exposed by an integration |
| `select.*` | A dropdown entity exposed by an integration - has a fixed set of options |
| `sensor.*` | A read-only value entity exposed by an integration or a template |
| Template sensor | A sensor whose value is calculated from a Jinja2 template - defined in `configuration.yaml` under `template:` |

---

## GoodWe hardware and integration terms

| Term | Meaning |
|---|---|
| EEPROM / flash | Non-volatile memory on the inverter. Survives power cycles. Operation mode changes from HA are written here. The exact chip type isn't publicly documented and community discussion uses "EEPROM" and "flash" interchangeably - we use "persistent storage" in the docs where the distinction doesn't matter. Typical write-cycle ratings for either type are 100,000+, which at HA's mode-change rate is likely more than the inverter's service life. The bigger reason to avoid HA-driven mode writes is that they overwrite the SolarGo TOU schedule slot, not the wear concern itself. |
| RAM register | Volatile memory on the inverter. Lost on power cycle or inverter restart. The experimental integration's EMS commands target RAM, which means no flash wear, but the inverter reverts to its last saved state if power is lost. |
| BMS | Battery Management System - the controller inside the battery pack that monitors cell voltages, temperatures, and SOC |
| SOC | State of Charge - battery level as a percentage (0-100%) |
| SOH | State of Health - battery degradation as a percentage (100% = new, lower = degraded) |
| LFP | Lithium Iron Phosphate - the battery chemistry used in GoodWe ESA batteries. Tolerant of regular 100% charging, unlike older NMC or Li-ion chemistries. Periodic full charges help calibrate the BMS. |
| DoD | Depth of Discharge - how far down the battery is allowed to discharge. A DoD of 90% means the battery discharges to 10% SOC minimum. |
| MPPT | Maximum Power Point Tracker - the circuit that extracts maximum power from each solar string at the optimal voltage |
| EMS | Energy Management System - GoodWe's name for the RAM-based mode/power control registers exposed in the experimental integration |
| Economic Mode / TOU | GoodWe's schedule-based charge/discharge mode. Called "Eco mode" in the HA integration. TOU (Time of Use) refers to the time-window scheduling within this mode. |
| Fast Charging | GoodWe's name for the forced grid-charge register - `switch.goodwe_fast_charging_switch`. This is a separate register from the Economic Mode schedule slots and does not conflict with them in the same way. |
| SolarGo | GoodWe's older consumer mobile app. Originally the primary end-user app, increasingly being superseded by SEMS+ as GoodWe migrates features across. SolarGo is still functional for everything covered in this guide (TOU schedules, Modbus TCP enable, monitoring), but new setups should prefer SEMS+. |
| SEMS+ | GoodWe's newer end-user app and installer portal in one. GoodWe is gradually moving consumer features into SEMS+ from SolarGo, so SEMS+ is the recommended app for most day-to-day use including TOU schedule setup. **Installer-level access uses a separate password to SolarGo** - even if you've set a custom installer password in SolarGo, SEMS+ wants its own; the two apps don't share access state. If you need installer-level features in SEMS+ and the SolarGo password isn't accepted, that's why. |
| Feed-in Limit | What SEMS+ calls the grid export power limit - the same entity as `number.goodwe_grid_export_limit` in HA. |

---

## Method 4 - what does what

Since Method 4 splits responsibility between the GoodWe app and HA, here is an explicit breakdown of which system owns each job:

| Job | Owned by | Why |
|---|---|---|
| Charge battery during free window (11-14) | GoodWe app (Economic Mode / TOU schedule) | Native firmware charges to 100% and holds. HA's `fast_charging_switch` exits forced-charge mode when the SOC target is met and the inverter reverts to self-consumption, which means the battery starts discharging into household load (including any EV that's plugged in) during the rest of the free window - the opposite of what you want when grid is free. TOU prevents that. |
| Hold battery at 100% after charge target is reached | GoodWe app (Economic Mode / TOU schedule) | Same reason as above |
| Check SOC before arming peak export | HA | Requires live sensor reading and conditional logic - the app cannot do this |
| Enforce export wattage limit during peak | HA (`number.goodwe_grid_export_limit`) | Dynamic - HA can block export entirely if SOC is too low, or restore it if a previous session left it at 0 |
| Report nightly profit | HA notification | Uses helper-stored rates and a session kWh delta calculation |
| Discharge battery during peak | GoodWe app (Economic Mode / TOU discharge schedule) or inverter self-consumption | Either works - a SEMS+ TOU discharge schedule is more explicit, but self-consumption will also draw the battery down during peak |
