# ⚡ GoodWe VPP Automations

This folder contains ready-to-use YAML automations. To use them, copy the YAML code and paste it into a new Home Assistant automation (using the "Edit in YAML" option). 

**Make sure to look for the `# EDIT:` tags in the code to swap in your specific GoodWe entity IDs.**

---

## 🛠️ Required Helpers for the "GloBird Zero Hero" Automation

The GloBird Zero Hero automation is highly advanced. It calculates your nightly profit and prevents the battery from draining into the grid if your State of Charge (SOC) is too low. 

To make this work, you must create two "Helpers" in Home Assistant before running the automation.

### 1. Create the Enable/Disable Toggle
This switch allows you to easily pause the automation if you go on holiday or want to temporarily disable peak exporting.
1. In Home Assistant, go to **Settings > Devices & Services > Helpers**.
2. Click **+ CREATE HELPER** and select **Toggle**.
3. Name it: `Zero Hero Enabled`
4. Choose an icon (e.g., `mdi:flash`).
5. Click Create. *(This will create the entity `input_boolean.zero_hero_enabled`)*.

### 2. Create the Export Math Helper
This stores the total number of kWh your inverter had exported at the exact moment the 6:00 PM peak started, allowing the automation to calculate how much you sold specifically during the peak window.
1. In Home Assistant, go to **Settings > Devices & Services > Helpers**.
2. Click **+ CREATE HELPER** and select **Number**.
3. Name it: `Zero Hero Export Start`
4. Set the **Minimum value** to `0` and **Maximum value** to something very high like `100000`.
5. Set **Step size** to `0.1`.
6. Click Create. *(This will create the entity `input_number.zero_hero_export_start`)*.

---

Once those two helpers are created, you are ready to copy and paste the `globird_zero_hero.yaml` code into your automations!
