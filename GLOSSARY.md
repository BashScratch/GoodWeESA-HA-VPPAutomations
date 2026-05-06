# Glossary - GoodWe / GloBird / Home Assistant Terminology

One of the most confusing things about setting up a GoodWe ESA with Home Assistant on a GloBird plan is that the same concept is called something different in every interface: the GoodWe hardware docs, the SolarGo/SEMS+ app, the native HA integration, the experimental HACS integration, and the GloBird plan documentation all have their own vocabulary. This glossary maps them together.

> **Verification note:** Terms in the HA integration columns are verified against the native HA integration source and the experimental integration's GitHub README. GoodWe app terminology is verified against GoodWe user manuals and community reports. Where exact option strings in the experimental integration could not be confirmed from primary sources, this is noted explicitly.

---

## Inverter operating modes

| What it does | GoodWe hardware docs | SolarGo / SEMS+ app | HA native integration (`select.goodwe_inverter_operation_mode`) | HA experimental integration |
|---|---|---|---|---|
| Normal self-consumption - use solar first, battery fills gaps, export surplus | Self-use Mode | Self-consumption / Self-use | `General mode` | `General mode` |
| Schedule-based charge or discharge using time windows (e.g. free window 11-2, peak window 6-8pm) | Economic Mode | Economic Mode / TOU schedule | `Eco mode` | `Eco mode` |
| Hold battery fully charged, pass grid to loads during outage | Backup Mode | Backup Mode | `Backup mode` | `Backup mode` |
| Force-charge battery right now, as fast as possible | Fast Charge / Forced Charge | Charge Now | `switch.goodwe_fast_charging_switch` + `number.goodwe_fast_charging_soc` | Same |
| Allow export up to a specific wattage limit | Export Power Limit / Feed-in Limit | Power Limit / Feed-in Limit | `number.goodwe_grid_export_limit` | `number.goodwe_grid_export_limit` |
| EMS - direct battery charge/discharge via RAM register command | EMS (Energy Management System) | Not exposed in app | Not available in native integration | `select.goodwe_ems_mode` + `number.goodwe_ems_power_limit` |

### The "Eco Mode" / "Economic Mode" / "TOU" naming tangle

This is the most common point of confusion, and it comes from GoodWe using the same concept under slightly different names in different contexts:

- **"Economic Mode"** is what GoodWe's hardware documentation and the SolarGo/SEMS+ app call the scheduled charge/discharge operating mode. This is the mode where you set time windows for when the battery should charge, discharge, or hold.

- **"TOU" (Time of Use)** is the *scheduling feature type* within Economic Mode - it refers to the time-window schedule itself, not the mode name. When a GoodWe manual or forum post says "set up a TOU schedule", they mean "create charge/discharge time windows inside Economic Mode".

- **"Eco Mode"** is what the Home Assistant native integration (and the experimental one) calls this same mode in `select.goodwe_inverter_operation_mode`. The option string is `Eco mode`.

So: **HA "Eco mode" = app "Economic Mode"** - and both use TOU-style time scheduling. They are the same thing under two names.

**Critical gotcha:** When HA writes an Eco Mode command, it writes directly into EEPROM slot 1, overwriting whatever Economic Mode schedule you previously set in the app. The app and HA are not aware of each other's schedule data - HA simply replaces it. This is why Method 3 avoids touching the operation mode from HA entirely.

### EMS modes (experimental integration only)

The experimental integration exposes `select.goodwe_ems_mode` with several options. The GoodWe integration README describes the following scenarios, though the **exact option strings as they appear in the HA select entity may vary by inverter model and firmware** - always verify against your actual entity in Developer Tools > States before building automations:

| Scenario | What it does |
|---|---|
| Auto (self-use) | Normal self-consumption - battery responds to household demand |
| Charge | Force battery to charge from PV (high priority) or grid (low priority) |
| Discharge / Export battery | Battery discharges; PV covers house load first, then battery exports surplus |
| Export grid (sell to grid) | Targets a specific export wattage; PV preferred, battery fills gap |
| Charge from grid | Force charge from grid (high priority); PV used if available |

The Method 2 automation in this guide uses `"Charge"`, `"Discharge"`, and `"Auto"`. **If these option strings don't match your entity, check Developer Tools > States > `select.goodwe_ems_mode` for the actual available options and adjust the YAML accordingly.** Common alternative names for the discharge-with-export scenario include `"Export AC"` and `"Export grid"`.

---

## GloBird Zero Hero tariff terminology

| What it means | GloBird's term | What this guide calls it |
|---|---|---|
| 11:00-14:00 period where grid electricity costs nothing | "Zero Import Window" / "Free Electricity Period" | Free window |
| 18:00-20:00 (or 21:00) period where export earns a premium | "Super Export Window" | Peak window |
| High export rate during peak (e.g. $0.15/kWh total) | "Super Export Credit" | Super rate |
| Export rate once the cap is exceeded (e.g. $0.06/kWh) | "Basic Export Credit" / "Feed-in tariff" | Base rate |
| Daily credit for not importing during the peak window | "Zero-Grid Credit" | Daily credit |
| kWh threshold above which super rate drops to base rate | "Super Export Cap" | Super cap (10 kWh older plan, 15 kWh newer plan) |

### Note on the super export rate structure

GloBird structures the super rate internally as: **base rate + bonus = total super rate** (e.g. $0.06 base + $0.09 bonus = $0.15 total). For the purposes of automation, you only care about the *total rate you receive* during peak ($0.15) vs what you receive once you've exceeded the cap ($0.06). The helpers in these automations store and use the total figures.

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
| EEPROM / flash | Non-volatile memory on the inverter. Survives power cycles. The HA native integration writes operation mode changes here. Has a limited number of write cycles (~100,000) - community wisdom flags this as a concern but no documented bricked-inverter cases exist, so treat it as a soft consideration rather than a hard limit. The bigger reason to avoid HA-driven mode writes is that they overwrite the SEMS+ TOU schedule slot. |
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
| SolarGo | GoodWe's primary mobile app for end users. Used for monitoring and basic settings. |
| SEMS+ | GoodWe's installer/advanced portal - has more settings than SolarGo, including detailed TOU schedule configuration. Access to some settings may be restricted by your installer. |
| Feed-in Limit | What SEMS+ calls the grid export power limit - the same entity as `number.goodwe_grid_export_limit` in HA. |

---

## Method 3 - what does what

Since Method 3 splits responsibility between the GoodWe app and HA, here is an explicit breakdown of which system owns each job:

| Job | Owned by | Why |
|---|---|---|
| Charge battery during free window (11-14) | GoodWe app (Economic Mode / TOU schedule) | Native firmware handles the 100%-and-hold behaviour correctly. HA's `fast_charging_switch` exits forced-charge mode when the SOC target is met, after which the inverter can revert to self-consumption and drain the battery. |
| Hold battery at 100% after charge target is reached | GoodWe app (Economic Mode / TOU schedule) | Same reason as above |
| Check SOC before arming peak export | HA | Requires live sensor reading and conditional logic - the app cannot do this |
| Enforce export wattage limit during peak | HA (`number.goodwe_grid_export_limit`) | Dynamic - HA can block export entirely if SOC is too low, or restore it if a previous session left it at 0 |
| Report nightly profit | HA notification | Uses helper-stored rates and a session kWh delta calculation |
| Discharge battery during peak | GoodWe app (Economic Mode / TOU discharge schedule) or inverter self-consumption | Either works - a SEMS+ TOU discharge schedule is more explicit, but self-consumption will also draw the battery down during peak |
