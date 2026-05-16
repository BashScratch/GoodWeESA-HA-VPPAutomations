# GoodWe VPP Template Sensors

Template sensors can't be set up via the HA UI - they need to live in your configuration files. This folder has the sensors that make the Zero Hero automations useful on a dashboard.

## What's here

- **`globird_zero_hero_sensors.yaml`** - two template sensors:
  - `sensor.zero_hero_session_export` - live kWh exported during the current peak window.
  - `sensor.zero_hero_session_profit` - live AUD earned during the current peak window, including the daily credit.
- **`goodwe_polarity_fix.yaml`** - template sensor that corrects the active-power sign on firmware revisions that report it inverted. Optional. See the file header for how to tell if you need it.
- **`goodwe_voltage_statistics.yaml`** - five `statistics` platform sensors that summarise grid voltage over 1-hour and 24-hour windows (min, max, mean). Pairs with [`../automations/advanced/grid_voltage_sag_alert.yaml`](../automations/advanced/grid_voltage_sag_alert.yaml) for the historical-graph half of the diagnostic picture.

The template sensors (`globird_zero_hero_sensors.yaml`, `goodwe_polarity_fix.yaml`) go under the `template:` integration. The statistics sensors (`goodwe_voltage_statistics.yaml`) go under the legacy `sensor:` key with `platform: statistics`. These are two different homes in your config - see the install notes inside each file.

The Zero Hero sensors update in real time and sit happily on a dashboard (I'd suggest a [mini-graph-card](https://github.com/kalkih/mini-graph-card) for the export, and a big number card for the profit).

## Install

### Step 1: Know your config structure

Open `configuration.yaml`. Look for a line like:

```yaml
template: !include templates.yaml
```

- If that line exists, you'll add the sensor code to `templates.yaml`.
- If it doesn't, you'll add a `template:` block directly into `configuration.yaml`.

### Step 2: Paste the code

Open [`globird_zero_hero_sensors.yaml`](./globird_zero_hero_sensors.yaml), copy the content, and paste it into the right file based on Step 1.

Watch the indentation. If you're pasting into `configuration.yaml` under a new `template:` heading, the `- sensor:` block should be indented two spaces. If you're pasting into `templates.yaml`, it lives at the root of the file.

### Step 3: Verify your GoodWe sensor names match

Before reloading, open **Developer Tools > States** and check that you have:

- `sensor.goodwe_total_energy_export` - lifetime export kWh counter
- `sensor.goodwe_battery_state_of_charge` - referenced by the Zero Hero automations alongside this file

These names have shifted across mletenay integration versions (older builds may have suffixes like `_2` or different wording). If your entity is named differently, update the references in `globird_zero_hero_sensors.yaml` - there's currently one `sensor.goodwe_total_energy_export` reference inside the template state expression. The templates fail silently to `0` if the sensor doesn't exist, so this is worth a one-minute check now to save a frustrating debug later.

### Step 4: Make sure the helpers exist

These sensors rely on helpers created for the automations. If you've already set up one of the Zero Hero automations per [the strategy guide](../automations/globird/README.md#required-home-assistant-helpers), you have them. Otherwise, create them first.

### Step 5: Reload

Go to **Developer Tools > YAML** and click **Check Configuration**. If it's green, click **Reload Template Entities**. If you're changing `configuration.yaml`, a full restart is safer.

Your two new sensors should appear under the `sensor` domain. Add them to a dashboard and enjoy watching the profit tick up during peak.

## Adding support for other providers

The structure of this folder is deliberately flat for now because we've only got one provider to deal with. If you've got template sensors for Amber, Energy Locals, or another plan and want to contribute, put them in a file named after the plan (`amber_shift_sensors.yaml` or similar) and update this README.
