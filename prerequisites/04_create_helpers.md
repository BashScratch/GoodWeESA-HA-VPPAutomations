# 4. Create the required helpers

**Helpers** are little user-defined entities - toggles, numbers, dates - that you create in the Home Assistant UI and that automations can read from and write to. Think of them as configurable knobs and switches that live with the rest of your HA entities.

We use them so that things like the GloBird tariff rates, the daily credit, and the SOC guard threshold are all editable from your dashboard rather than hardcoded into YAML. When GloBird changes a rate, you change a number in the UI; you don't open the YAML editor.

## What helpers do you need?

The full list lives in the strategy guide at **[automations/globird/README.md](../automations/globird/#required-home-assistant-helpers)**. We don't duplicate it here because there's a six-of-them shared list plus method-specific extras, and keeping the canonical list in one place stops the two from drifting apart.

This guide covers the **how**. The strategy guide covers the **what**.

## Step 1 - Open the Helpers page

In Home Assistant: **Settings > Devices & services > Helpers** (the **Helpers** tab is along the top, next to "Integrations" and "Devices").

If you've never used helpers before, the page will be empty.

## Step 2 - Create a helper

Click the **Create helper** button (bottom right). HA will offer you a list of helper types. The two we use:

- **Toggle** - for `input_boolean.zero_hero_enabled` and `input_boolean.zero_hero_force_safe`. A simple on/off switch.
- **Number** - for everything else (rates, caps, SOC thresholds, the export tracker). A numeric value with a min, max, and step size.

For each helper in the strategy guide's list, click **Create helper**, pick the right type, and fill in the fields:

- **Name** - exactly what the strategy guide says (e.g. "Zero Hero Enabled"). HA will derive the entity ID from this. **Use the exact name** so the entity ID matches what the YAML expects.
- **Icon** - optional, but a quick `mdi:flash` or `mdi:currency-usd` makes the dashboards readable at a glance.
- **Min value, Max value, Step size** - copy from the strategy guide. These prevent you from typing nonsense values into the dashboard later.
- **Initial value** - the starting value. The strategy guide tells you what to set each helper to.
- **Unit of measurement** - optional, but helps the dashboard render nicely (e.g. `AUD/kWh`, `%`, `kWh`).

Click **Create**.

## Step 3 - Verify the entity ID

After creating each helper, click into it and confirm the entity ID matches what the strategy guide says.

For example, a toggle named "Zero Hero Enabled" should have entity ID `input_boolean.zero_hero_enabled`. If HA created it with a slightly different ID (because the name was different, or there's a conflict), edit the helper and fix the entity ID - the YAML automations will only find the helper if the entity ID is exactly right.

You can also rename the entity ID directly: click the helper, then the cog icon, then the Entity ID field.

## Step 4 - Add them to a dashboard

Strictly optional, but recommended: build yourself a small "Zero Hero Settings" card on your HA dashboard with all the helpers grouped together. When GloBird adjusts the super rate or the daily credit, you'll thank yourself for being able to change it in two clicks.

A simple [Entities card](https://www.home-assistant.io/dashboards/entities/) listing all the helpers does the job.

## Common gotchas

- **The export tracker (`input_number.zero_hero_export_start`) is managed by the automation** - don't change its value yourself. It stores the kWh-export-so-far number at 18:00 each day so the automation can calculate how much you exported during peak. Set it to 0 initially and leave it alone.
- **Min/max bounds matter.** If you set the Super Rate helper with max=1 and then try to set it to 1.50, HA will silently clamp it to 1.0. Set the maxes generously.
- **The unit field is cosmetic.** It changes how the dashboard displays the value but doesn't affect the YAML logic. Set it for readability.
- **Don't accidentally create both `input_number.zero_hero_rate_super` and `input_number.zero_hero_rate_super_2`.** This happens if you create a helper, delete it, and re-create it with the same name - HA sometimes appends `_2` to avoid a conflict with the lingering deleted one. Verify the entity ID after each create. If you get `_2` suffixes, edit them out via the cog icon.

## What you should have at the end of this guide

All the helpers from the strategy guide created and visible on the Helpers page, with the right entity IDs. If you've got any with `_2` suffixes, fix them now - debugging "the automation isn't reading my rates" later is much harder than catching it here.

Next: [Find your actual entity IDs in Developer Tools](./05_find_your_entities.md).
