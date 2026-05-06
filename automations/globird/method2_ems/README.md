# Method 2: EMS RAM Commands (experimental / advanced)

> **Experimental.** Method 2 depends on a community-maintained HACS integration that pokes registers the official integration doesn't expose. A future GoodWe firmware update could change those registers and silently break this method. If you want a quiet life, use [Method 3](../method3_hybrid/) instead.

> Before you copy anything: read the [strategy guide](../) to understand the tradeoffs and create the required helpers. For a breakdown of EMS mode options, see the [Glossary](../../../GLOSSARY.md).

This method sidesteps flash entirely by writing to the Energy Management System registers in RAM. The operation mode is never touched, so any SolarGo/SEMS+ TOU schedules you've set are preserved. The EMS discharge mode is genuinely clever - it covers your house load first and then targets the specified export wattage.

## Known limitations

- **EMS mode option strings vary by firmware.** The YAML uses `"Charge"`, `"Discharge"`, and `"Auto"`, but your inverter may report `"Export AC"`, `"Self-use"`, or other variants. **Check Developer Tools > States > `select.goodwe_ems_mode` and update the YAML to match your actual options before saving.** This is the most common reason Method 2 silently does nothing.
- **HA crash recovery is handled by a watchdog**, but verify it works on your install. The automation includes a `time_pattern` watchdog (every 5 minutes) that returns the inverter to `"Auto"` if it's stuck in a forced mode outside the active windows. Test it: during a peak window, flip `input_boolean.zero_hero_force_safe` ON and confirm the inverter returns to Auto within ~5 minutes.
- **Do not run Method 2 alongside Method 3.** Method 2 itself is safe (it never changes operation mode), but if you ever experimentally enable Method 1 *or* manually switch to Eco/General mode while testing, **the inverter deletes your SEMS+ TOU schedule** - the schedule that Method 3 relies on for the free-window charge ([Whirlpool 9n111qlk](https://forums.whirlpool.net.au/thread/9n111qlk)). Pick one method per inverter and stick with it.
- **Modbus TCP must be enabled on the inverter.** It's off by default on the GoodWe ESA. If you can't see EMS entities at all (or the integration won't connect), this is the most likely cause. The SolarGo app's communication settings is where you turn it on; [Whirlpool 9xv6wp84](https://forums.whirlpool.net.au/thread/9xv6wp84) has community walkthroughs for the various app versions.

## What you need

- **The GoodWe Experimental integration** by mletenay, installed via HACS:
  https://github.com/mletenay/home-assistant-goodwe-inverter
  The native HA GoodWe integration does not expose the EMS entities. This method won't work without the experimental one.
- The entities `select.goodwe_ems_mode` and `number.goodwe_ems_power_limit` - check your actual setup in Developer Tools > States.
- The eight helpers listed in the [strategy guide](../#required-home-assistant-helpers). Method 2 uses two extras beyond the six shared ones: `input_boolean.zero_hero_force_safe` (panic switch - flip ON to force inverter back to Auto via the watchdog) and `input_number.zero_hero_min_export_soc` (SOC floor for arming peak export).
- A notification device set up in HA.

## Install

1. Confirm the experimental integration is working. Go to **Developer Tools > States** and search for `ems` - you should see the EMS mode select and power limit number entities.
2. **Check the actual option strings** in `select.goodwe_ems_mode`. Click the entity in Developer Tools and look at the available options. The YAML uses `"Charge"`, `"Discharge"`, and `"Auto"` - if your entity shows different strings, update the YAML to match before saving.
3. If HACS is not installed, install it first: https://hacs.xyz/
4. Open [`globird_ems_control.yaml`](./globird_ems_control.yaml) and copy the whole file.
5. In HA, go to **Settings > Automations & Scenes**, create a new empty automation, click the three dots > **Edit in YAML**, paste.
6. Replace every `# EDIT:` placeholder with your real entity IDs and notify service name.
7. Save, enable, and watch the first cycle carefully.
8. **Watchdog test:** during a peak window, flip `input_boolean.zero_hero_force_safe` to ON. Within 5 minutes the inverter should return to Auto and you should see a "Zero Hero Watchdog" notification. Flip it back off when satisfied.

## Watch out for

- **EMS mode option strings.** Most common failure point - see "Known limitations" above.
- **RAM vs EEPROM.** EMS commands target RAM. If the inverter loses power or restarts during the peak window, it reverts to its last EEPROM-saved state. Usually fine (it'll go back to General/self-consumption mode), but the automation won't know this happened.
- **Peak end time.** Default is 21:00. If your plan ends at 20:00, change both the `peak_export_end` trigger and the watchdog's `21:02:00` time check.
- **Super export cap.** The YAML profit calculation reads from `input_number.zero_hero_super_cap`. Set this helper to `10` (older plan) or `15` (newer plan).
