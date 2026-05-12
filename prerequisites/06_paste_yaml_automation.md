# 6. How to paste a YAML automation into Home Assistant

This is the bit that nobody warns you about. Every guide on the internet says "paste the YAML" and then waves at the HA UI. The HA UI hides the YAML editor by default, so if you've never done this before, you can spend half an hour clicking around and not finding it.

Here's exactly how to do it.

## What you're about to do

Each YAML file in this repo is one Home Assistant **automation** - a self-contained piece of logic that says "when X happens, do Y". You're going to:

1. Create a new, empty automation in HA.
2. Switch the editor from visual mode to YAML mode (this is the hidden bit).
3. Paste the contents of the YAML file from this repo.
4. Find every `# EDIT:` comment and replace the placeholder with your real entity ID.
5. Save the automation.

You'll repeat this for each automation you're installing - typically 3 or 4 of them depending on which method you picked.

## Step 1 - Open the Automations page

In Home Assistant: **Settings > Automations & Scenes**.

You'll see a list of any existing automations. Don't worry about those.

## Step 2 - Create a new empty automation

Click **Create Automation** (bottom right). HA will offer you three options:

- "Create new automation"
- "Use a blueprint"
- "Start with empty automation"

**Pick "Start with empty automation."** This is critical - the other two options try to be helpful and put you in a different editor mode that fights with what we're doing.

You'll land on a screen with a visual editor that has empty fields for triggers, conditions, and actions. We're about to throw all of that away.

## Step 3 - Switch the editor to YAML mode (the hidden bit)

Look at the **top right corner** of the automation editor screen. You'll see a three-dots icon (sometimes called a "kebab menu"). Click it.

A dropdown appears with several options. Click **"Edit in YAML"**.

The screen swaps from a visual editor to a plain text box. You're now in YAML mode.

> **First time?** HA might show a warning: "Switching to YAML mode will discard your current changes." Click "yes" - there's nothing to lose because we haven't entered anything yet.

## Step 4 - Clear the default text and paste

The YAML editor will have a default skeleton in it - something like:

```yaml
alias: New automation
description: ""
trigger: []
condition: []
action: []
mode: single
```

**Select all (Ctrl+A or Cmd+A) and delete it.** Now the editor is empty.

Open the YAML file from this repo (in your browser, in a text editor, whatever's easiest) and copy the entire contents - every line from the top to the bottom. Paste it into the empty HA editor.

The pasted YAML should start with `alias: "GoodWe ESA: ..."` (or `alias: "Energy: ..."` for the EV reminder, or `alias: "System: ..."` for the time sync and battery alert).

## Step 5 - Find and replace every `# EDIT:` comment (DO THIS BEFORE SAVING)

Use **Ctrl+F (or Cmd+F)** to search inside the editor. Search for `EDIT`.

> **Important: do all your EDIT replacements before you click Save.** Home Assistant strips all YAML comments (anything after a `#`) when it saves an automation. So once you hit Save, the `# EDIT:` markers are gone forever and you've lost your roadmap. If you save with placeholders still in there and come back later, you'll be scanning the whole YAML by eye looking for what to change. Make every replacement in one pass while the comments are still visible.

Every match is a line where you need to replace a placeholder with a real value from your install. The most common ones:

> **Tip:** open a second browser window or tab pointed at **Developer Tools > States** while you do this. Search for `goodwe` in that tab. As you work through each `# EDIT:` line in the YAML editor, flick over and copy-paste the real entity ID rather than retyping it. Typos in entity IDs are the most common reason an automation appears to save fine but does nothing at runtime.

> **A note on the `| float(0)` pattern.** You'll see things like `{{ states('sensor.goodwe_battery_state_of_charge') | float(0) }}` scattered throughout the YAML in this repo. The `| float(0)` filter converts the sensor's value to a number; the `(0)` argument is the fallback if the value can't be converted (sensor unavailable, returning `unknown`, etc.). Without it, the template crashes the whole automation when the inverter blips offline for a poll cycle. **If you write your own additions or modifications, keep this pattern - it's defensive code that costs nothing and prevents silent failures.** Same logic applies to `| int(0)` for integers.

| Placeholder | Replace with |
|---|---|
| `select.goodwe_inverter_operation_mode` | Your real operation mode select entity (might have `_2` suffix) |
| `number.goodwe_eco_mode_power` | Your real Eco Mode power number entity |
| `select.goodwe_ems_mode` | Your real EMS mode select entity |
| `number.goodwe_ems_power_limit` | Your real EMS power limit number entity |
| `number.goodwe_grid_export_limit` | Your real export limit number entity |
| `sensor.goodwe_battery_state_of_charge` | Your real SOC sensor entity |
| `sensor.goodwe_total_energy_export` | Your real total export sensor entity |
| `notify.mobile_app_your_device_name` | Your real notify service from [Guide 03](./03_install_companion_app.md) |
| `binary_sensor.ev_charger_vehicle_connected` | Your EV charger's "vehicle connected" binary sensor (EV reminder only) |
| `button.goodwe_synchronize_inverter_clock` | Your real clock-sync button entity (time sync only) |

You should have your entity IDs written down from [Guide 05](./05_find_your_entities.md). Open that list now and work through every `# EDIT:` line, replacing the example with your real value.

You can leave the `# EDIT: ...` comment text in place after replacing while you're still editing - it's just a comment, HA ignores it during this stage. The moment you save, however, HA strips all the comments out, so they won't survive to "future-you" anyway. Plan to make every replacement in this one editing pass.

For Method 3 only: also check the option strings (`"Auto"`, `"Charge"`, `"Discharge"`) match what your `select.goodwe_ems_mode` exposes. If your inverter reports `"Export AC"` instead of `"Discharge"`, change the YAML to match.

## Step 6 - Save

Click **Save** in the bottom right of the editor. HA will validate the YAML - if there's a syntax error, you'll get a red error bar at the top with the line number. Common causes:

- **Indentation broken**: YAML is whitespace-sensitive. If your editor (or your terminal) replaced spaces with tabs, the indentation breaks. Re-paste from a plain text editor.
- **Stray characters at the start**: smart quotes, an extra blank line at the very top, a UTF-8 BOM. Delete anything before the `alias:` line.
- **Missing `# EDIT:` replacement**: HA generally won't complain about an entity that doesn't exist, but the automation will silently fail at runtime. Always grep for `EDIT` before saving.

If save succeeds, the automation is now stored in HA but hasn't run yet.

## Step 7 - Enable (or don't, if you want to test first)

By default, a saved automation is **enabled**. You can see this on the Automations page - there's a toggle switch next to each automation.

For the Zero Hero automations, **we recommend disabling `input_boolean.zero_hero_enabled` first** (the helper, not the automation) before you let the day's first cycle run. That way the automation will trigger but won't do anything dramatic - it'll just exit at the first condition check. You can watch the trigger fire in the HA logs without the inverter changing state. Enable the helper once you've confirmed the triggers fire correctly. The [first-run checklist](./09_first_run_checklist.md) walks through this.

## Step 8 - Repeat for each YAML

You'll typically install 3 or 4 automations:

- The method-specific one (Method 2 / 2 / 3 from `automations/globird/`)
- `goodwe_battery_fault_alert.yaml` (recommended)
- `goodwe_time_sync.yaml` (recommended)
- `ev_free_window_reminder.yaml` (only if you've got an EV)

Each one is a separate automation in HA. Don't try to paste them all into a single one.

## Common mistakes

- **Forgot to switch to YAML mode.** You opened the empty automation, started typing in the visual editor, got confused. Go back and use the three-dots menu > Edit in YAML.
- **Pasted into "Edit Dashboard" by accident.** That's a different editor. You want **Settings > Automations & Scenes > Create Automation**, not the dashboard editor.
- **Save button greyed out.** Usually means there's a YAML syntax error. The save button reactivates when the YAML parses cleanly.
- **Automation appears in the list but nothing fires.** Check that the toggle next to it is on (blue). And check the **Trace** (click the automation, then "Traces" tab) - HA records every trigger fire, even ones that exited via a condition.

## How to disable an automation if it misbehaves

Go to **Settings > Automations & Scenes**, find the row, click the toggle to off. The automation stays in HA but stops firing. You can re-enable any time.

For an emergency stop on Zero Hero specifically, flipping `input_boolean.zero_hero_enabled` to off (via your dashboard or the helpers page) will short-circuit every Zero Hero action without you needing to disable individual automations.

## What you should have at the end of this guide

- All the automations you're installing pasted, edited with your real entity IDs, and saved.
- A successful test save (no red error bar) for each.
- Optionally: `input_boolean.zero_hero_enabled` disabled while you watch the first day's triggers fire harmlessly.

Next: [How to add template sensors to configuration.yaml](./07_add_template_sensors.md). Or, if you're skipping the dashboard sensors, jump to [Guide 08 (Method 4 only)](./08_sems_tou_schedule.md) or [Guide 09: First-run checklist](./09_first_run_checklist.md).
