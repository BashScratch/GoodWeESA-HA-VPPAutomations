# 7. How to add template sensors to configuration.yaml

The dashboard sensors in this repo (the ones that show "how much have I exported in the current peak window" and "how much have I made tonight") are **template sensors**. Unlike automations, template sensors can't be created from the HA UI - they have to live in your configuration files. This guide is the editing process.

If you don't care about dashboard sensors and just want the automations running, you can skip this whole guide. The automations work without the sensors.

## What you need

You need a way to edit text files inside HA's config directory. The three common options:

- **File Editor add-on** (easiest, HA OS / Supervised installs) - Settings > Add-ons > Add-on Store > search "File editor" > install. Gives you a file editor inside the HA web UI.
- **Studio Code Server add-on** (also easiest, full VS Code in your browser) - same install path, but the add-on is called "Studio Code Server".
- **SSH or Samba** if you've got those configured. Then edit with your favourite text editor on your laptop.

If you're on **HA Container** or **HA Core**, you don't have add-ons - edit the files via SSH, Samba, or whatever method you use to access the config directory directly.

Pick one. The rest of this guide assumes you can open and save files inside HA's config directory.

## Step 1 - Find your config directory

The HA config directory contains `configuration.yaml`, `automations.yaml`, and a bunch of other files. Common locations:

- **HA OS / Supervised:** `/config/` (visible from add-ons), or `/usr/share/hassio/homeassistant/` on the host.
- **HA Container:** wherever you mounted as `/config` in your Docker run command.
- **HA Core (Python venv install):** typically `~/.homeassistant/`.

Open `configuration.yaml` from this directory.

## Step 2 - Decide where the template sensors go

Look at your `configuration.yaml`. There are two common scenarios:

### Scenario A - Your `configuration.yaml` has a line like:

```yaml
template: !include templates.yaml
```

If you see this, your template sensors live in a separate file called `templates.yaml`. Open `templates.yaml` (it might be empty, or have other template entries). Skip to Step 3.

### Scenario B - Your `configuration.yaml` has no `template:` section at all (or has an inline one).

You'll add a new `template:` section directly to `configuration.yaml`. Skip to Step 4.

## Step 3 - Paste into `templates.yaml`

Open `templates.yaml`. Open [`sensors/globird_zero_hero_sensors.yaml`](../sensors/globird_zero_hero_sensors.yaml) from this repo and copy the **entire contents**.

Paste at the **bottom** of `templates.yaml`. Don't change indentation - the `- sensor:` block in the file is already at the correct indentation level for inclusion in `templates.yaml`.

Find the `# EDIT:` comment in the pasted block and replace `sensor.goodwe_total_energy_export` with your real total-export sensor entity ID (from [Guide 05](./05_find_your_entities.md)).

Save the file.

## Step 4 - Paste into `configuration.yaml`

If you don't have a `template: !include` line, paste the sensors directly into `configuration.yaml`. The structure needs to look like this:

```yaml
# ... your existing configuration.yaml ...

template:
  - sensor:
      - name: "Zero Hero Session Export"
        unique_id: zero_hero_session_export
        # ... rest of the sensor definition ...
      - name: "Zero Hero Session Profit"
        unique_id: zero_hero_session_profit
        # ... rest ...
```

Note that when pasting into `configuration.yaml` directly under a `template:` heading, **the `- sensor:` block must be indented two spaces** to sit correctly under `template:`. The file in this repo is structured for the `templates.yaml` include scenario; if you're in the inline scenario, indent everything two extra spaces.

Replace the `# EDIT:` placeholder with your real total-export sensor.

Save the file.

> **Tip:** if you've got an existing `template:` block in `configuration.yaml` already, append the `- sensor:` block to it rather than creating a second `template:` heading. YAML will silently use only the second one if you have two top-level `template:` keys.

## Step 5 - Validate the configuration

In HA: **Developer Tools > YAML** (the YAML tab). Click **Check Configuration** (the big button).

- Pro: **Configuration valid** - proceed.
- Con: **Invalid configuration** - HA will show the error and the file/line. Most common: indentation off by a level, or stray smart quotes from copy-pasting via a rich-text app. Fix and retry.

Don't restart HA on a broken config. It won't start cleanly.

## Step 6 - Reload templates

Once the config is valid:

- If you put the sensors in `templates.yaml`: **Developer Tools > YAML > Reload Template Entities**. Click it. The new sensors appear without restarting HA.
- If you put them directly in `configuration.yaml`: a full **HA restart** is safer. Settings > System > Restart.

## Step 7 - Verify the sensors exist

Go to **Developer Tools > States** and search for `zero_hero`. You should see:

- `sensor.zero_hero_session_export` - value should be `0.0` outside peak; will show real kWh during peak.
- `sensor.zero_hero_session_profit` - same; shows AUD during peak.

If the sensors show `unavailable`, the most common cause is the `# EDIT:` placeholder for `sensor.goodwe_total_energy_export` is wrong. Fix and reload.

## Step 8 - Add to a dashboard

The whole point. Add the sensors to a dashboard card:

- A simple [Entities card](https://www.home-assistant.io/dashboards/entities/) listing both sensors gives you the running totals on screen.
- A [Mini Graph Card](https://github.com/kalkih/mini-graph-card) (HACS) plotted against `sensor.zero_hero_session_export` gives you a live graph of the export rate during peak.
- A big [Markdown Card](https://www.home-assistant.io/dashboards/markdown/) showing `Profit so far: $ {{ states('sensor.zero_hero_session_profit') }}` is satisfying during peak.

## What you should have at the end of this guide

- Template sensors added to either `templates.yaml` or `configuration.yaml`.
- `# EDIT:` placeholder replaced with your real total-export sensor.
- HA config validates clean.
- Both `sensor.zero_hero_session_export` and `sensor.zero_hero_session_profit` visible in Dev Tools > States.

Next: [SEMS+ TOU schedule (Method 3 only)](./08_sems_tou_schedule.md) or skip to [First-run checklist](./09_first_run_checklist.md).
