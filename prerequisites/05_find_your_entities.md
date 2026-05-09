# 5. Find your actual entity IDs in Developer Tools

Every YAML file in this repo has `# EDIT:` comments next to entity IDs. Those entity IDs are *examples* - they're what most installs end up with, but yours might be subtly different. This guide is how you find what you've actually got.

## Why entity IDs differ between installs

The GoodWe HA integration auto-generates entity IDs based on the inverter's reported model and serial number. Things that can shift the IDs:

- **You re-added the integration.** If you've ever installed and removed the GoodWe integration (or had it crash and re-set itself up), HA holds onto the original entity IDs for the deleted instance and appends `_2` to the fresh ones. So you end up with `sensor.goodwe_battery_state_of_charge_2` as the live entity and `sensor.goodwe_battery_state_of_charge` as a dead leftover. Repeated install/remove cycles can produce `_3`, `_4` and so on. The fix is to remove the dead instance properly via Settings > Devices & services and rename the live entity (cog icon > Entity ID) to drop the suffix - or just live with the suffix and use whichever entity ID is actually receiving data.
- **You're running both the native and experimental integrations.** They mostly produce non-overlapping entities, but a few (like SOC sensor) appear in both, and one of them gets the `_2`.
- **Your inverter model name affects entity naming.** ESA inverters generally produce sensors named with the `goodwe_` prefix, but in older firmware some names included the model number (e.g. `sensor.goodwe_etn_battery_soc`).

## Step 1 - Open Developer Tools > States

In Home Assistant: click **Developer Tools** in the sidebar (the spanner/screwdriver icon, near the bottom). Then click the **States** tab along the top.

You'll see a long list of every entity in your install. There's a search box at the top.

## Step 2 - Search for `goodwe`

Type `goodwe` in the search box. The list filters down to your inverter-related entities. Expect to see something like 30-80 entities, depending on which integrations you've installed.

## Step 3 - Identify the entities each method needs

For each method, here's the entity checklist. Find each one in your list, click it to confirm it has a real value (not "unavailable" or empty), and note the exact entity ID for when you edit the YAML.

### All methods need

| Purpose | Likely entity ID | Notes |
|---|---|---|
| Battery state of charge (%) | `sensor.goodwe_battery_state_of_charge` | Should show 0-100 with `%` unit |
| Battery temperature (°C) | `sensor.goodwe_battery_temperature` | Used by the BMS fault alert automation |
| Battery error code | `sensor.goodwe_battery_error` | Usually shows `0` or empty when fine |
| Battery warning code | `sensor.goodwe_battery_warning` | Same |
| Battery state of health (%) | `sensor.goodwe_battery_state_of_health` | Should show ~100 on a new battery |
| Total energy exported (lifetime kWh) | `sensor.goodwe_total_energy_export` | Counter that only increases |
| Synchronise inverter clock | `button.goodwe_synchronize_inverter_clock` | A button entity, used by the time sync automation |

### Method 2 (Standard Eco Mode) extras

| Purpose | Likely entity ID |
|---|---|
| Operation mode | `select.goodwe_inverter_operation_mode` |
| Eco Mode power | `number.goodwe_eco_mode_power` |

When you find `number.goodwe_eco_mode_power`, **click it** and look at its **Attributes** in the right pane. Note the `min` and `max` values:

- If `min` is `-100` and `max` is `100`, the integration uses **percentage**.
- If `min` is `-5000` and `max` is `5000` (or larger), the integration uses **watts**.

This tells you what to set `input_number.zero_hero_eco_power` to. The Method 2 YAML has a UNIT-TRAP comment block at the top with detailed instructions.

### Method 3 (EMS RAM Commands) extras

| Purpose | Likely entity ID |
|---|---|
| EMS mode | `select.goodwe_ems_mode` |
| EMS power limit | `number.goodwe_ems_power_limit` |

When you find `select.goodwe_ems_mode`, **click it** and look at its **Attributes**, specifically the `options` list. The exact strings vary - typical examples:

```yaml
options:
  - "Auto"
  - "Charge"
  - "Discharge"
  - "Export AC"
```

Note exactly what your install reports. The Method 3 YAML uses `"Auto"`, `"Charge"`, and `"Discharge"`. If your inverter shows `"Export AC"` instead of `"Discharge"`, edit the YAML to match before saving.

### Method 4 (Hybrid) extras

| Purpose | Likely entity ID |
|---|---|
| Grid export power limit | `number.goodwe_grid_export_limit` |
| Fast charging switch | `switch.goodwe_fast_charging_switch` |

For `number.goodwe_grid_export_limit`, click and confirm the value isn't `unavailable`. On some installs the value shows as `0`; on others it shows the inverter's nameplate maximum (e.g. `5000` or `10000`). Either is fine - it just tells you what the inverter is currently limited to.

## Step 4 - Handle the `_2` suffix problem

If you've got `sensor.goodwe_battery_state_of_charge` **and** `sensor.goodwe_battery_state_of_charge_2`, you've got two integration instances claiming the same entity. One of them is dead (no value, last updated days ago); the other is live.

Click each in turn. Look at the **Last updated** timestamp in the right-hand pane. The one that's updated within the last few minutes is the live one - use that entity ID in the YAML.

If you can't tell which is live (both look the same age), you've probably got both the native and experimental integrations installed and both are happily polling. That's fine - pick the one that matches the rest of your entity names. If most of your entities don't have `_2`, use the unsuffixed `_2`-free versions; if most of them do have `_2`, use the suffixed ones.

To tidy this up properly: go to **Settings > Devices & services**, click into each GoodWe integration, and disable the one you're not using. The redundant entities will eventually disappear. (Note: don't delete the integration outright unless you're sure - you can always re-enable a disabled integration.)

## Step 5 - Write your entity IDs down

Make a list. You'll need this when you paste each YAML in [Guide 06](./06_paste_yaml_automation.md).

A simple text file or sticky note works. The exact IDs you need depend on which method you picked, but the "All methods" table above is a minimum.

## What you should have at the end of this guide

- A confirmed, written list of the entity IDs your install actually uses for each thing the YAML references.
- A note of whether your `goodwe_eco_mode_power` is in **percentage** or **watts** (Method 2 only).
- A note of the exact `select.goodwe_ems_mode` option strings your inverter exposes (Method 3 only).
- No `_2` confusion left over - you know which copy of each entity is live.

Next: [How to paste a YAML automation into Home Assistant](./06_paste_yaml_automation.md).
