# Method 1: Standard Eco Mode

> Before you copy anything: read the [strategy guide](../) to understand the tradeoffs and create the required helpers. If you're unsure what "Eco Mode" means vs "TOU" vs "Economic Mode", see the [Glossary](../../../GLOSSARY.md).

The straightforward, single-automation approach. HA toggles the inverter between Eco Mode (for forced charge/discharge) and General Mode.

## Limitations worth knowing about

**Charging tops out at ~10kW.** GoodWe ESA inverters can charge at up to 13.5kW when the firmware orchestrates 10kW AC + 3.5kW DC simultaneously, but HA can only command the AC side, so this method is capped at the AC limit (~10kW). If you want the full 13.5kW combined charge rate during the GloBird free window, use Method 3, which leans on a SolarGo TOU schedule for charging.

**The 5kW peak target is *total inverter discharge*, not grid-export specifically.** Method 1 sets `number.goodwe_eco_mode_power` to 5000W (or 100% if percentage-mode), which discharges the battery at 5kW total. House load comes out of that 5kW first, the rest goes to the grid. If your house load drops mid-peak, more goes to the grid; if it spikes, less does. Method 3's `number.goodwe_grid_export_limit` is precise grid-export control by comparison.

**This overwrites any TOU schedule you've set in the SolarGo app** every time it changes mode. If you use SolarGo for scheduling anything, use Method 3 instead.

## What you need

- **The experimental GoodWe Inverter integration by mletenay**, installed via HACS: <https://github.com/mletenay/home-assistant-goodwe-inverter>. Method 1 uses `number.goodwe_eco_mode_power` to set the charge/discharge magnitude, and that entity is only exposed by the experimental integration (not the native one). Earlier versions of this guide said Method 1 was "no HACS required" - that's wrong, sorry.
- The entities `select.goodwe_inverter_operation_mode` and `number.goodwe_eco_mode_power` from your install. Check Developer Tools > States; the entity names may have a `_2` suffix if you've installed and removed the integration before (HA reserves the original ID for the deleted instance and appends `_2` to the fresh one).
- The eight helpers listed in the [strategy guide](../#required-home-assistant-helpers). Method 1 uses two extras beyond the six shared ones: `input_number.zero_hero_eco_power` (Eco Mode magnitude - read the unit-trap note in the YAML) and `input_number.zero_hero_min_export_soc` (SOC floor for arming peak export).
- A notification device set up in HA.

## Install

1. Open [`globird_eco_mode.yaml`](./globird_eco_mode.yaml) and copy the whole file.
2. In HA, go to **Settings > Automations & Scenes**, click **Create Automation**, skip the template picker, choose **Start with an empty automation**.
3. Click the three-dots menu in the top right > **Edit in YAML**. Paste.
4. Use `Ctrl+F` to find every `# EDIT:` comment. Replace placeholders with your real entity IDs and notify service name.
5. Save, enable, and watch the next free window and peak window play out.

## Watch out for

- **Eco Mode Power units (the unit-trap).** Some integration versions expect a percentage (`-100` to `100`), others expect Watts (`-5000` to `5000`). The YAML reads from `input_number.zero_hero_eco_power` so you set this once in the UI rather than editing YAML, but you still need to know which unit your integration uses. Check Developer Tools > States > `goodwe_eco_mode_power` and look at the entity's `min`/`max` attributes; the YAML header explains exactly what to do. If the automation runs but nothing happens at the inverter, this is the first thing to verify.
- **Peak end time.** The YAML defaults to 21:00. Older GloBird plans end at 20:00. Adjust the trigger time accordingly.
- **Super export cap.** The YAML profit calculation reads from `input_number.zero_hero_super_cap`. Set it to `10` (older plan) or `15` (newer plan).
- **First run.** Disable `input_boolean.zero_hero_enabled` first, run through a full day to see that the charge and mode changes fire without anything going sideways, then re-enable for the peak logic to kick in.
- **Don't run Method 1 alongside Method 3.** Method 1 toggles operation mode, which deletes the SolarGo TOU schedule that Method 3 depends on ([Whirlpool thread "GoodWe ESA - Setting export TOU with SOC limit"](https://forums.whirlpool.net.au/thread/9n111qlk)). Pick one.
