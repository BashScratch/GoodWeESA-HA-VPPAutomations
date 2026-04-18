# Method 1: Standard Eco Mode

This folder contains the YAML automation for the Standard Eco Mode method. 

This approach uses the official, native Home Assistant GoodWe integration to change the inverter's operating behaviour. It does not require installing any experimental third-party integrations via HACS.

## How the Standard Method Works

The GoodWe integration exposes an entity called `Operation Mode` (usually `select.inverter_operation_mode`). This allows Home Assistant to switch the inverter from its default "General Mode" (which strictly covers house load) into "Eco Mode" (which allows forceful charging and discharging).

* **The Free Window (11:00 AM):** The automation switches the inverter to Eco Mode and sets the `Eco Mode Power` to a negative value (forcing the battery to charge from the grid).
* **The Afternoon (2:00 PM):** It returns the inverter to General Mode to run off the newly stored solar and grid power.
* **The Peak Window (6:00 PM):** It switches back to Eco Mode and sets the `Eco Mode Power` to a positive value, commanding the battery to forcefully export power to the grid to capture the GloBird Super Export rate. 
* **The Night (9:00 PM):** It reverts to General Mode to cover the house load until morning.

### The Trade-offs

While this method is straightforward and uses native entities, it has significant hardware and software impacts:
* **Flash Memory Wear:** Every time Home Assistant changes the Operation Mode or the Eco Mode Power, the integration writes those commands to the inverter's EEPROM (Flash Memory). Executing this automation will consume at least four flash memory write cycles per day.
* **SEMS+ Overwriting:** To execute Eco Mode commands, Home Assistant commandeers the very first Time of Use (TOU) slot in the inverter's registers. Doing this instantly erases any existing charging or discharging schedules you have programmed into your GoodWe SEMS+ mobile app. If you use this automation, Home Assistant must become the sole controller of your inverter's schedule.

## Prerequisites

1. A working connection to your inverter using the official Home Assistant GoodWe integration.
2. Ensure your integration exposes the `select.operation_mode` and `number.eco_mode_power` entities. 
3. The two Home Assistant Helpers detailed in the main GloBird Strategy Guide (`input_boolean.zero_hero_enabled` and `input_number.zero_hero_export_start`).

## Implementation

1. Open `globird_eco_mode.yaml`.
2. Copy the YAML code.
3. In Home Assistant, navigate to **Settings > Automations & Scenes**, and create a new automation.
4. Click the three dots in the top right corner and select **Edit in YAML**, then paste the code.
5. Use `Ctrl+F` to find the `# EDIT:` comments. Replace the placeholder entities with your exact GoodWe Operation Mode and Eco Mode Power entity IDs, as well as your Home Assistant notification device name.
6. **Important Note:** Check your specific GoodWe integration to see if `Eco Mode Power` expects a percentage (e.g., `-100` to charge, `100` to discharge) or raw Watts (e.g., `-5000` to charge, `5000` to discharge) and update the values in the YAML accordingly.
7. Save and monitor the first few cycles.
