# 9. First-run checklist

Inverter reachable, HA talking to it, helpers created, automations pasted, sensors showing values. Before you turn the whole thing loose, run through this checklist. It catches the silly mistakes that aren't worth a debugging session at 18:01 with a half-discharged battery.

## Pre-flight checklist

Tick each line before enabling. If any of these are "uh, not sure", go back to the relevant guide and confirm.

### Hardware

- [ ] The inverter is reachable from HA (Settings > Devices & services > GoodWe shows live data, not "Failed to connect").
- [ ] Inverter clock is roughly synchronised with HA (within a couple of minutes). The [time sync automation](../automations/goodwe_time_sync.yaml) handles this once installed; before that you can check the time displayed on the inverter's local web interface.
- [ ] Battery SOC sensor (`sensor.goodwe_battery_state_of_charge` or your equivalent) is showing a real number, not `unavailable`.
- [ ] Total export sensor is showing a real number that increases over time.

### Helpers

- [ ] All helpers from the [strategy guide](../automations/globird/README.md#required-home-assistant-helpers) exist with the right entity IDs (no `_2` suffixes).
- [ ] `input_boolean.zero_hero_enabled` exists and is currently **off**.
- [ ] `input_number.zero_hero_export_start` is at `0` (not some leftover value from testing).
- [ ] Tariff helpers (`rate_super`, `rate_base`, `rate_credit`, `super_cap`) are set to your actual GloBird plan values, not the example numbers.

### Automations

- [ ] The method-specific automation (Method 2, 3, or 4 - whichever you picked) is pasted, EDIT-replaced, and saved.
- [ ] `goodwe_battery_fault_alert.yaml` is pasted (highly recommended).
- [ ] `goodwe_time_sync.yaml` is pasted.
- [ ] All automations show up in Settings > Automations & Scenes with their toggles **enabled** (the toggle next to the name).
- [ ] No syntax errors when you saved any of them.

### Notifications

- [ ] Companion app installed on at least one phone, logged in, notify service confirmed.
- [ ] `notify.mobile_app_your_device_name` replaced with your real notify service name in every automation that uses it (search every YAML for `notify.mobile_app`).
- [ ] Test notification fired successfully from Developer Tools > Actions.

### Method 4 only

- [ ] SEMS+ TOU schedule is set up (11:00-13:55, charge, 100%, grid priority - the 13:55 end is deliberate, see prereq 08).
- [ ] Inverter is in Economic Mode (the working mode dropdown in SEMS+).
- [ ] **No automations** touching `select.goodwe_inverter_operation_mode`.

## First-day plan

Don't enable `input_boolean.zero_hero_enabled` on day one. Watch first.

### 11:00 - free window starts

- **Method 2 or 3:** the automation will trigger but will see `zero_hero_enabled` is off and exit on the condition check (or it'll force-charge anyway because Method 2's free-window block doesn't gate on the toggle - check the YAML before deciding). Watch the trace: **Settings > Automations & Scenes**, click the automation to open the editor, then click the **Traces icon** in the top right of the editor window. You'll see the trigger fired and what condition path it took. (Older HA versions had this as a "Traces" tab on the automation details page.)
- **Method 4:** The SEMS+ TOU schedule fires. The HA automation isn't involved in the free window. Watch the battery SOC climb in the GoodWe integration sensors or the SEMS+ app. Should hit 100% before 14:00; if it doesn't, your inverter's charging slower than expected - usually because solar is producing very little (cloudy day) and the inverter is grid-only at the 10kW AC ceiling.

### 14:00 - free window ends

- Battery should be at or near 100%.
- **Method 2 or 3:** automation switches inverter back to General / Auto mode.
- **Method 4:** inverter naturally exits the TOU charge window because it's past 14:00. SOC stays high.

### 17:56 - pre-peak guard check

This is the moment that matters. The automation evaluates SOC - Method 4 against its stepped bracket table (which decides *how hard* to export), Methods 2 and 3 against the guard threshold (`input_number.zero_hero_min_export_soc`).

- If SOC is high enough, you should get a "Zero Hero Armed" notification (Method 4's includes the bracket rate it picked) - but only if `zero_hero_enabled` is on. With it off, the automation exits silently.
- Watch the trace either way. Confirm the trigger fired at 17:56 and the condition path went where you expected.

### 18:00-21:00 (or 20:00) - peak window

- With `zero_hero_enabled` off, nothing happens. The inverter behaves normally - solar covers what it can, battery covers the rest, grid imports if needed.
- With it on, the export limit is set to the armed rate (Method 4's bracket pick, or whatever you've configured on Methods 2/3) and the battery starts emptying into the grid. Watch the export sensor climb.

### Peak end

- Export limit restored to unrestricted (10000W in the YAML default).
- "Zero Hero Complete" notification fires if `zero_hero_enabled` is on, with the night's profit.
- Battery SOC should be down to whatever the natural discharge endpoint was.

## When to enable `input_boolean.zero_hero_enabled`

After at least **one full day** of watching the system run with the toggle off, with no surprises in the traces. Confirm:

- All triggers fired at the times you expected.
- No automation errored out (red items in the Logbook).
- The inverter behaved sensibly during free and peak windows even without HA's intervention.

Then flip the toggle on, and watch day two carefully. Now HA will arm the export limit at 18:00.

## Bail-out switches

Three layers of "stop", from softest to hardest:

1. **Flip `input_boolean.zero_hero_enabled` off** - every Zero Hero action gates on this. Inverter goes back to its native behaviour. SEMS+ TOU schedule (Method 4) keeps running.
2. **Flip `input_boolean.zero_hero_force_safe` on** (Method 3 only) - within 5 minutes the watchdog returns the inverter to Auto regardless of any other state.
3. **Disable individual automations** - Settings > Automations & Scenes, toggle off. Stops triggers from firing entirely.

For an actual emergency (inverter doing something visibly wrong, e.g. exporting at full whack when it shouldn't), the fastest fix is usually the inverter's own DC isolator switch. Power-cycling the inverter clears any stuck RAM-level state. Use this only if needed.

## What to watch over the first week

- **Daily profit notifications**: do the numbers match what GloBird credits you? Their app's "today's earnings" view is the cross-check.
- **The free-window charge** (Method 4): SOC should reliably hit 100% before 14:00. If it consistently misses, your TOU schedule isn't quite right or your inverter is undersized for your battery.
- **Peak window depth**: how low does the battery go? On Method 4, if the floor guard is firing on ordinary nights, the brackets are too hot for your battery or the floor is too high - re-cut one of them (the YAML's comment block has the math). On Methods 2/3, if it ends each peak below your guard threshold (`input_number.zero_hero_min_export_soc`), the export target is too high or the threshold too low. Tune.
- **Rate changes**: GloBird sometimes adjusts plan rates. Check your latest bill or plan documents and update the helpers if the figures have changed.
- **Inverter clock drift**: some GoodWe units drift several minutes per week. The `goodwe_time_sync.yaml` automation runs daily at 03:00 to correct this. If you're seeing your TOU schedule fire at the wrong times, check the inverter clock.

## What you should have at the end of this guide

A working setup that you understand and can debug. If you can answer "what does each automation do, and how do I turn it off if it misbehaves?" without looking, you're sorted.

Welcome to the dynamic-pricing battery game. Goodnight, and may your peak windows always be sunny.
