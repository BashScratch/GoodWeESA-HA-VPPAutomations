# Method 2: Power User EMS (RAM Commands)

This folder contains the YAML automation for the Energy Management System (EMS) control method. 

This approach is considered the "power user" method because it bypasses the standard GoodWe Home Assistant integration and communicates directly with the inverter's temporary RAM.

## How the EMS Method Works

Standard Home Assistant commands (like changing to Eco Mode) write to the inverter's Flash Memory (EEPROM), which overwrites your SEMS+ app schedules and causes hardware wear over time.

This method avoids that entirely by manipulating two specific EMS registers:
1. **EMS Mode** (`select.inverter_ems_mode`)
2. **EMS Power Limit** (`number.inverter_ems_power_set`)

During the GloBird evening peak (6:00 PM), the automation changes the EMS Mode to `Export AC` and sets the Power Limit to `5000` (Watts). The GoodWe internal logic automatically prioritises covering your house load first, and then pushes exactly 5000W of the remaining power directly to the grid. 

At the end of the peak window, it reverts the EMS Mode to `Auto`, returning the inverter to its standard self-consumption behaviour. 

### The Benefits
* **Zero Flash Wear:** Because these commands are written to RAM, they do not consume flash memory write cycles.
* **Preserves SEMS+:** It does not overwrite or erase any Time of Use (TOU) schedules you have configured in the GoodWe SEMS+ app.
* **Intelligent Export:** By using `Export AC`, the inverter inherently balances the house load against the export limit, ensuring you do not accidentally pull power from the grid to meet your export target.

### The Risks and Downsides
* **Unofficial Integration:** This method relies on an experimental, community-maintained custom integration. A future GoodWe firmware update could change the EMS registers and break the integration without warning.
* **Connection Dependency:** Because EMS commands are active real-time instructions, they require a stable connection. If your Home Assistant server crashes or loses Wi-Fi while the `Export AC` command is active, the inverter will not receive the `Auto` command at the end of the window. It may get stuck continuously exporting until you manually intervene or restart the inverter.

## Prerequisites

To use this automation, you must install the experimental GoodWe integration via the Home Assistant Community Store (HACS). The standard GoodWe integration natively included in Home Assistant does not expose the EMS entities.

1. Install [HACS](https://hacs.xyz/) if you have not already.
2. Install the **GoodWe Experimental** integration by GitHub user `mletenay` ([Repository Link](https://github.com/mletenay/home-assistant-goodwe-inverter)).
3. Configure the integration with your inverter's IP address.

## Implementation

1. Open `globird_ems_control.yaml`.
2. Copy the YAML code.
3. In Home Assistant, navigate to **Settings > Automations & Scenes**, and create a new automation.
4. Click the three dots in the top right corner and select **Edit in YAML**, then paste the code.
5. Use `Ctrl+F` to find the `# EDIT:` comments. You must replace the placeholder entities with your exact EMS Mode and EMS Power Limit entity IDs, as well as your Home Assistant notification device name.
6. Save and monitor the first few evening cycles to ensure the EMS commands are engaging and disengaging correctly.
