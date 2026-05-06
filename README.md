# GoodWe ESA - Home Assistant VPP Automations

Home Assistant YAML, template sensors, and strategy guides for GoodWe ESA-series inverters connected to Virtual Power Plants or dynamic wholesale energy plans.

Current focus: **GloBird Zero Hero**. The repo is structured so other providers (Amber Electric, Energy Locals, and friends) can slot in cleanly when guides for them are written.

This is a work in progress. Things will change. Read the disclaimer at the bottom before wiring any of this into a live inverter.

---

## Why this exists

Automating a GoodWe ESA is not as simple as flicking a switch. How you send commands to the inverter matters - the wrong approach can silently overwrite the TOU schedule you painstakingly set up in the GoodWe SEMS+ app, or quietly cap your free-window charging at 10kW when the inverter could be doing 13.5kW.

So there isn't one automation here. There are three. They trade off differently on charge throughput, simplicity, and how much you trust an experimental HACS integration not to break on the next firmware update. Pick the one that matches your tolerance for fiddling and risk.

---

## The three methods, at a glance

| Method | Approach | Free-window charge rate | SEMS+ schedules | Stability | Recommended for |
|---|---|---|---|---|---|
| **Method 1: Standard Eco Mode** | Native HA integration toggles Eco Mode on/off | ~10kW (AC only) | **Overwrites them** | Stable, but limited | People who want the simplest setup and don't use SEMS+ schedules |
| **Method 2: EMS RAM Commands** | Experimental HACS integration pokes RAM registers | ~10kW (AC only) | Preserved | Depends on community integration | Power users comfortable with unofficial integrations |
| **Method 3: Hybrid General Mode** | SEMS+ handles timing, HA enforces limits | **~13.5kW (AC + DC blended)** | Preserved | Rock solid | **Most people. This is what we recommend.** |

Full explanation of each, and why Method 3's throughput advantage settles the recommendation: see **[automations/globird/](./automations/globird/)**.

---

## What's in this repo

```
.
├── README.md                                 <- you are here
├── LICENSE                                   <- MIT
├── automations/
│   ├── ev_free_window_reminder.yaml          <- nudge yourself to plug in the EV
│   ├── goodwe_battery_fault_alert.yaml       <- critical BMS alerts on your phone
│   ├── goodwe_time_sync.yaml                 <- keeps the inverter clock honest
│   └── globird/
│       ├── README.md                         <- strategy guide + helpers setup
│       ├── method1_standard/
│       ├── method2_ems/
│       └── method3_hybrid/
└── sensors/
    ├── README.md
    └── globird_zero_hero_sensors.yaml        <- live profit tracking on your dashboard
```

### General automations (recommended regardless of which method you pick)
- **[EV Free Window Reminder](./automations/ev_free_window_reminder.yaml)** - pings you at 10:45 if the EV isn't plugged in yet, so you don't miss the free window.
- **[GoodWe Time Sync](./automations/goodwe_time_sync.yaml)** - syncs the inverter clock to HA once a day. Drift is the quiet killer of TOU schedules.
- **[GoodWe Battery Fault Alert](./automations/goodwe_battery_fault_alert.yaml)** - fires a critical notification if the BMS reports an error or warning. Your battery failing quietly is not the vibe.

### Provider-specific
- **[GloBird Zero Hero strategy + automations](./automations/globird/)** - the three methods and when to use which.

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

The GoodWe integration landscape is a bit of a tip. The standard integration (built into Home Assistant) and the experimental one (by mletenay, via HACS) use slightly different default entity names. Your actual entities may also be suffixed with `_2` or similar if the integration has been reinstalled.

Every `# EDIT:` marker in the YAML is a place where you should double-check the entity exists in your setup before trusting it. Go to **Developer Tools > States** in HA and search for `goodwe` to see what you actually have.

If you're using Method 2, you specifically need the experimental integration - the standard one doesn't expose the EMS entities.

---

## Contributing, issues, suggestions

Open an issue on GitHub. If you're on a different VPP plan and want to add guides, PRs welcome - the structure under `automations/` is deliberately provider-scoped so new providers can land in their own folder.

---

## Disclaimer

These automations are provided as-is under the MIT [LICENSE](./LICENSE). They're designed to be sensible, but they send commands to expensive hardware that sits on your grid connection. Always test while watching what actually happens. I'm not liable for surprise grid charges, confused batteries, hardware wear, or your inverter developing an attitude.

Test it on a day you're home. Watch the first cycle. Read the logs. If something feels wrong, disable the automation and work out why before re-enabling.
