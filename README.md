# GoodWe ESA - Home Assistant VPP Automations

Home Assistant YAML, template sensors, and strategy guides for GoodWe ESA-series inverters connected to Virtual Power Plants or dynamic wholesale energy plans.

Current focus: **GloBird Zero Hero**. The repo is structured so other providers (Amber Electric, Energy Locals, and friends) can slot in cleanly when guides for them are written.

This is a work in progress. Things will change. Read the disclaimer at the bottom before wiring any of this into a live inverter.

---

## Why this exists

Automating a GoodWe ESA is not as simple as flicking a switch. How you send commands to the inverter matters - the wrong approach can silently overwrite the TOU schedule you painstakingly set up in the GoodWe SolarGo app, or quietly cap your free-window charging at 10kW when a single-phase 10kW ESA could be doing 13.5kW (per GoodWe's own [ESA Series single-phase datasheet, V2.1 April 2026](https://admin.goodwe.com/Api/downloadFile?id=3448&mid=60&type=2) - "Max. Charging Power" of 13.5kW for the GW9.999K-EHA-G20). Note that this throughput advantage is single-phase only; three-phase ESAs (model ending in ETA-G20) cap max charging at nominal AC and don't get the AC+DC blending bump - see the strategy guide's [model compatibility section](./automations/globird/#model-and-firmware-compatibility).

So there isn't one automation here. There are four methods - one app-only baseline (no HA at all) and three HA-driven approaches that build on top of it. They trade off differently on charge throughput, simplicity, and how much you trust an experimental HACS integration not to break on the next firmware update. Pick the one that matches your tolerance for fiddling and risk.

## Why use HA at all? (acknowledging GoodWe's apps)

A fair question. GoodWe is actively shipping firmware and app updates - the 13.5kW combined AC+DC charging capability has been rolling out via firmware from around April 2026, although availability is uneven (some users have it on the standard OTA, others have had to ask Level 2 support for a standalone firmware push, and some recent firmware releases have shipped without it - it's a mishmash). Their L2 support is responsive when contacted directly. They've also been gradually migrating consumer features from the older **SolarGo** app into the newer **SEMS+** app, so we recommend SEMS+ for new setups (SolarGo still works for everything covered in this guide if you're already comfortable there). A lot of what HA adds here (precise grid-export control during peak, conditional SOC guards, profit notifications keyed to your tariff structure) is reasonably likely to land in SEMS+ itself eventually. This repo is filling a gap that exists today, not staking out a permanent moat. If GoodWe ships those features in the app, by all means use them - the goal is the energy-bill outcome plus the niceties (notifications, profit reporting, dashboards), not the HA setup as an end in itself.

Until then, the smart layer this repo provides (read live SOC, decide whether to arm peak export, set a precise grid-export wattage, calculate nightly profit, fire notifications you can act on) adds to what SolarGo and SEMS+ already do. That's the case for HA in this specific project.

---

## The four methods, at a glance

| Method | Approach | Needs HA? | Free-window charge rate | SEMS+ schedules | Recommended for |
|---|---|---|---|---|---|
| **Method 1: App-only** | Two TOU slots in SEMS+, nothing else | No | Up to 13.5kW (single-phase 10kW model) | Created in app, owned by app | Simplest possible setup, or first-time users learning TOU |
| **Method 2: Standard Eco Mode** | HA toggles Eco Mode on/off | Yes (HACS) | ~10kW (AC only) | **Overwritten** | People who want HA in charge and don't care about SEMS+ schedules |
| **Method 3: EMS RAM Commands** | HA pokes EMS RAM registers | Yes (HACS) | ~10kW (AC only) | Preserved | Dynamic-pricing VPPs (Amber etc) where mode changes happen often |
| **Method 4: Hybrid General Mode** | SEMS+ TOU + HA enforces limits | Yes (HACS optional) | **~13.5kW (AC + DC blended, single-phase 10kW model)** | Preserved | **Most people on Zero Hero. This is what we recommend.** |

Full explanation of each, and why Method 4's throughput advantage settles the recommendation: see **[automations/globird/](./automations/globird/)**.

---

## What's in this repo

```
.
├── README.md                                 <- you are here
├── LICENSE                                   <- MIT
├── GLOSSARY.md                               <- terminology mapping (GoodWe/SolarGo/HA/GloBird)
├── automations/
│   ├── ev_free_window_reminder.yaml          <- nudge yourself to plug in the EV
│   ├── goodwe_battery_fault_alert.yaml       <- critical BMS alerts on your phone
│   ├── goodwe_time_sync.yaml                 <- keeps the inverter clock honest
│   ├── globird/                              <- GloBird Zero Hero strategy + automations
│   │   ├── README.md                         <- strategy guide + helpers setup
│   │   ├── method1_app_only/                 <- Method 1 (app-only baseline)
│   │   ├── method2_standard/                 <- Method 2 (HA, standard Eco Mode)
│   │   ├── method3_ems/                      <- Method 3 (HA, EMS RAM commands)
│   │   ├── method4_hybrid/                   <- Method 4 (Hybrid, recommended)
│   │   └── zero_grid_credit_watchdog.yaml    <- daily-credit protection automation
│   └── advanced/                             <- optional advanced layer
│       ├── README.md
│       ├── grid_voltage_soak.yaml
│       ├── inverter_thermal_management.yaml
│       └── lfp_calibration_charge.yaml
├── prerequisites/                            <- start-here guides for first-time setup
│   ├── README.md
│   └── 01-09 numbered guides
└── sensors/
    ├── README.md
    ├── globird_zero_hero_sensors.yaml        <- live profit tracking on your dashboard
    └── goodwe_polarity_fix.yaml              <- corrects sign convention on affected firmware
```

### General automations (recommended regardless of which method you pick)
- **[EV Free Window Reminder](./automations/ev_free_window_reminder.yaml)** - pings you at 10:45 if the EV isn't plugged in yet, so you don't miss the free window.
- **[GoodWe Time Sync](./automations/goodwe_time_sync.yaml)** - syncs the inverter clock to HA once a day. If the inverter's clock drifts away from HA's (and the rest of the world's), TOU schedules start firing at the wrong times and HA's triggers stop matching the inverter's behaviour. Drift is the quiet killer of TOU schedules.
- **[GoodWe Battery Fault Alert](./automations/goodwe_battery_fault_alert.yaml)** - fires a critical notification if the BMS reports an error or warning. Your battery failing quietly is not the vibe.

### Provider-specific
- **[GloBird Zero Hero strategy + automations](./automations/globird/)** - the four methods (one app-only, three HA-driven) and when to use which.

### Optional advanced automations
- **[Advanced automations](./automations/advanced/)** - protective and quality-of-life additions (grid-voltage soaking, inverter thermal management, LFP calibration tracker, polarity-fix template sensor). Optional layer on top of any of the four methods.

### Dashboard sensors
- **[Zero Hero live sensors](./sensors/)** - template sensors for "how much have I exported in the current peak window" and "how much have I made tonight".

---

## How to use this

1. Read the [GloBird strategy guide](./automations/globird/) and pick a method.
2. Create the required helpers (detailed in the strategy guide).
3. Copy the YAML from the method folder you picked.
4. In Home Assistant, go to **Settings > Automations & Scenes**, create a new automation, click the three dots, choose **Edit in YAML**, and paste.
5. Use `Ctrl+F` to find every `# EDIT:` comment in the pasted code. Replace the placeholder entity IDs with the real ones from your own HA setup. This step is not optional. Skipping it means the automation will do nothing, or worse, do something unexpected to an entity that happens to share a name.
6. Save, enable, and watch the first few days of runs to make sure it behaves.

---

## About entity naming

There are two GoodWe integrations for Home Assistant: the **native** one bundled with HA (read sensors, basic mode control), and the **experimental HACS** one by [mletenay](https://github.com/mletenay/home-assistant-goodwe-inverter). The HACS integration is a **layer over the native one**, not a replacement - it adds the active-control entities the native integration doesn't expose (Eco Mode power, EMS, fast-charging switch) while leaving the native sensors and entities in place. Per mletenay's README, you can uninstall HACS at any time and the native integration takes over seamlessly with your existing entity history preserved. The mletenay integration is well-maintained and ships fast updates when GoodWe firmware shifts; treat it as production-quality with the standard "experimental" caveat that it pokes registers GoodWe doesn't officially document.

Your actual entities may also be suffixed with `_2` (or `_3`, `_4`...) if you've installed and removed an integration before. HA reserves the original entity ID for the deleted instance and appends a number to the fresh one. So if the YAML examples reference `sensor.goodwe_battery_state_of_charge` and your install shows `sensor.goodwe_battery_state_of_charge_2`, that's the cause.

Every `# EDIT:` marker in the YAML is a place where you should double-check the entity exists in your setup before trusting it. Go to **Developer Tools > States** in HA and search for `goodwe` to see what you actually have.

**The active-control entities live in the HACS integration, not the native one.** Specifically:

- `number.goodwe_eco_mode_power` - the magnitude for forced charge/discharge in Eco Mode. **Method 2 needs this.**
- `switch.goodwe_fast_charging_switch` - force-charge from grid right now. **Method 4's midnight reset uses this** as a safety net (wrapped in `continue_on_error: true`, so Method 4 itself runs fine native-only - the line just no-ops if the entity isn't there).
- `select.goodwe_ems_mode` and `number.goodwe_ems_power_limit` - direct RAM-level mode and power. **Method 3 needs both.**

So Methods 1 and 2 require HACS. Method 4 will run on the native integration alone, but installing HACS too is recommended (no harm, and you get the safety-net line working).

## A concept worth understanding before you start: "discharge power" vs "grid export"

In SolarGo's TOU settings, **"discharge power" is total inverter output**, not grid-export specifically. A 10% setting on a 10kW inverter means 1kW total (house load + grid combined), not 1kW to the grid. This trips up a lot of people setting up VPP automations.

This is why Method 4 splits responsibility the way it does: the SolarGo TOU discharge slot is set to 100% (give the inverter full headroom), and HA's `number.goodwe_grid_export_limit` is the precise lever that actually controls grid export. Methods 1 and 2 use SolarGo-style "total discharge" semantics and inherit the imprecision.

---

## Contributing, issues, suggestions

Open an issue on GitHub. If you're on a different VPP plan and want to add guides, PRs welcome - the structure under `automations/` is deliberately provider-scoped so new providers can land in their own folder.

---

## If this saved you time (optional)

If this guide helped and you're not on GloBird yet, you can sign up via my referral link and we both get $50 credit: <https://quote.globirdenergy.com.au/quote?pcode=refer&ref=O5BX1X>. Completely optional - the guide and all the YAML in this repo work exactly the same whether you use the link or not. No pressure.

---

## Disclaimer

These automations are provided as-is under the MIT [LICENSE](./LICENSE). They're designed to be sensible, but they send commands to expensive hardware that sits on your grid connection. Always test while watching what actually happens. I'm not liable for surprise grid charges, confused batteries, hardware wear, or your inverter developing an attitude.

Test it on a day you're home. Watch the first cycle. Read the logs. If something feels wrong, disable the automation and work out why before re-enabling.
