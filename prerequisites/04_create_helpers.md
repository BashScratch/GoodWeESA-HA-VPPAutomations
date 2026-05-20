# 4. Create the required helpers

**Helpers** are little user-defined entities - toggles, numbers, dates - that you create in the Home Assistant UI and that automations can read from and write to. Think of them as configurable knobs and switches that live with the rest of your HA entities.

We use them so that things like the GloBird tariff rates, the daily credit, and the SOC guard threshold are all editable from your dashboard rather than hardcoded into YAML. When GloBird changes a rate, you change a number in the UI; you don't open the YAML editor.

## What helpers do you need?

The full list lives in the strategy guide at **[automations/globird/README.md](../automations/globird/README.md#required-home-assistant-helpers)**. We don't duplicate it here because there's a six-of-them shared list plus method-specific extras, and keeping the canonical list in one place stops the two from drifting apart.

This guide covers the **how**. The strategy guide covers the **what**.

## Step 1 - Open the Helpers page

In Home Assistant: **Settings > Devices & services > Helpers** (the **Helpers** tab is along the top, next to "Integrations" and "Devices").

If you've never used helpers before, the page will be empty.

## Step 2 - Create a helper

Click the **Create helper** button (bottom right). HA will offer you a list of helper types. We use three of them:

- **Toggle** - for `input_boolean.zero_hero_enabled` and `input_boolean.zero_hero_force_safe`. A simple on/off switch.
- **Number** - for the rates, caps, SOC thresholds, and the export tracker. A numeric value with a min, max, and step size. This covers almost all the helpers in the strategy guide.
- **Utility Meter** - only needed if you're installing the [Zero-Grid Credit watchdog](../automations/globird/zero_grid_credit_watchdog.yaml). Resets daily at 18:00 and tracks accumulated grid imports during the peak window. Step 6 below covers the setup.

For each helper in the strategy guide's list, click **Create helper**, pick the right type, and fill in the fields:

- **Name** - exactly what the strategy guide says (e.g. "Zero Hero Enabled"). HA will derive the entity ID from this. **Use the exact name** so the entity ID matches what the YAML expects.
- **Icon** - optional, but a quick `mdi:flash` or `mdi:currency-usd` makes the dashboards readable at a glance.
- **Min value, Max value, Step size** - copy from the strategy guide. These prevent you from typing nonsense values into the dashboard later.
- **Unit of measurement** - optional, but helps the dashboard render nicely (e.g. `AUD/kWh`, `%`, `kWh`).

Click **Create**.

> **Note: there is no "Initial value" field in the UI.** It used to be requested but Home Assistant only exposes it through YAML configuration, not the helper-creation dialog ([HA input_number docs](https://www.home-assistant.io/integrations/input_number/): *"`initial` is only available in a YAML configuration and not via the Home Assistant user interface"*). You'll set the starting value in the next step, after the helper exists.

## Step 3 - Set the starting value

A freshly-created number helper defaults to its minimum value (or `0` if min is `0`). Set it to whatever the strategy guide recommends:

- **Easiest:** on the Helpers page, click the helper, then use the slider or number-box on its details page to type the recommended starting value. The change saves automatically.
- **Alternative:** add the helper to a Lovelace dashboard card (see Step 5 below) and set the value from there. The same persistence applies.
- **Power-user route:** Developer Tools > Actions > `input_number.set_value`, target your helper, set the value. Useful if you want to script a bulk initial setup.

Whichever route you pick, HA persists the value across restarts via the state machine, so you only need to do this once per helper.

For toggle helpers (`input_boolean`), there's no starting-value step at all - they default to off after creation, which is the desired state for the master enable / force-safe toggles in this guide.

## Step 4 - Verify the entity ID

After creating each helper, click into it and confirm the entity ID matches what the strategy guide says.

For example, a toggle named "Zero Hero Enabled" should have entity ID `input_boolean.zero_hero_enabled`. If HA created it with a slightly different ID (because the name was different, or there's a conflict), edit the helper and fix the entity ID - the YAML automations will only find the helper if the entity ID is exactly right.

You can also rename the entity ID directly: click the helper, then the cog icon, then the Entity ID field.

## Step 5 - Add them to a dashboard

Strictly optional, but recommended: build yourself a small "Zero Hero Settings" card on your HA dashboard with all the helpers grouped together. When GloBird adjusts the super rate or the daily credit, you'll thank yourself for being able to change it in two clicks.

A simple [Entities card](https://www.home-assistant.io/dashboards/entities/) listing all the helpers does the job.

## Step 6 - Create the Utility Meter helper (only if installing the Zero-Grid Credit watchdog)

The Zero-Grid Credit watchdog ([`automations/globird/zero_grid_credit_watchdog.yaml`](../automations/globird/zero_grid_credit_watchdog.yaml)) needs a Utility Meter helper to track grid imports during the 18:00-21:00 peak window. Without this helper the watchdog will trigger on a sensor that doesn't exist and never fire. Skip this step if you're not installing that automation.

From the Helpers page, click **Create helper** > pick **Utility Meter** > fill in:

| Field | Value |
|---|---|
| **Name** | `Zero Hero Peak Grid Import` (HA will derive entity ID `sensor.zero_hero_peak_grid_import`) |
| **Input sensor** | `sensor.goodwe_total_energy_import` (or your equivalent - check Developer Tools > States if you have a `_2` suffix on this sensor) |
| **Meter reset cycle** | `Daily` |
| **Cycle offset (days/hours/minutes)** | `0 days, 18 hours, 0 minutes` so it resets at the start of the peak window |
| **Tariffs** | Leave empty - this isn't a multi-tariff meter, just a daily-resetting accumulator |
| **Periodically reset** | On (default) |

Click **Create**.

After creation you'll have `sensor.zero_hero_peak_grid_import` on the States page. Outside the peak window it should read `0` (or close to it - it accumulates from 18:00 each day and zeroes again at the next 18:00). During peak, it climbs as you import.

The watchdog automation also needs `input_number.zero_hero_credit_alert_threshold` (a regular Number helper, default value `25`, unit `Wh`) - create it via the Number flow in Step 2 the same way as the other helpers. The watchdog's YAML header has the full details and tuning guidance for both.

If you'd rather configure via `configuration.yaml` instead of the UI, the equivalent YAML block is:

```yaml
utility_meter:
  zero_hero_peak_grid_import:
    source: sensor.goodwe_total_energy_import   # EDIT: your import sensor
    cycle: daily
    offset:
      hours: 18
```

## Common gotchas

- **The export tracker (`input_number.zero_hero_export_start`) is managed by the automation** - don't change its value yourself. It stores the kWh-export-so-far number at 18:00 each day so the automation can calculate how much you exported during peak. Set it to 0 initially and leave it alone.
- **Min/max bounds matter.** If you set the Super Rate helper with max=1 and then try to set it to 1.50, HA will silently clamp it to 1.0. Set the maxes generously.
- **The unit field is cosmetic.** It changes how the dashboard displays the value but doesn't affect the YAML logic. Set it for readability.
- **Don't accidentally create both `input_number.zero_hero_rate_super` and `input_number.zero_hero_rate_super_2`.** This happens if you create a helper, delete it, and re-create it with the same name - HA sometimes appends `_2` to avoid a conflict with the lingering deleted one. Verify the entity ID after each create. If you get `_2` suffixes, edit them out via the cog icon.

## What you should have at the end of this guide

All the helpers from the strategy guide created and visible on the Helpers page, with the right entity IDs. If you've got any with `_2` suffixes, fix them now - debugging "the automation isn't reading my rates" later is much harder than catching it here.

Next: [Find your actual entity IDs in Developer Tools](./05_find_your_entities.md).
