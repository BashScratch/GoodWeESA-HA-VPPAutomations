# GoodWe ESA - Home Assistant VPP Automations

Welcome. This repository is a collection of Home Assistant YAML automations, guides, and template sensors designed specifically for GoodWe ESA series inverters.

If you are connected to a Virtual Power Plant (VPP) or a dynamic wholesale energy plan, configuring your inverter to charge and discharge at the right times is critical to maximising your return. 

**Note on Scope:** Currently, the primary focus and available guides in this repository are built around the **GloBird Zero Hero** plan. However, the repository has been structured modularly to allow for future VPP automations (such as Amber Electric or Energy Locals) to be added easily as they are developed.

---

## Understanding GoodWe Automations

Automating a GoodWe ESA is not as simple as just flicking a switch. Depending on how you send commands to the inverter via Home Assistant, you can inadvertently overwrite your GoodWe SEMS+ app settings or cause unnecessary hardware wear on the inverter's Flash Memory (EEPROM).

Because of this, we do not just provide a single automation. For complex VPP plans, we provide several different configuration methods ranging from standard Home Assistant commands to experimental RAM commands. This allows you to choose the setup that best balances profit, battery health, and system longevity.

Detailed explanations of these methods and the hardware limitations are included inside each provider's specific folder.

---

## Available Guides & Automations

### Provider-Specific VPP Automations
*   **[GloBird Zero Hero Automations & Strategy Guide](./automations/globird/)**
    *   Contains the complete technical guide and YAML files for Method 1 (Eco Mode), Method 2 (EMS RAM Commands), and Method 3 (Hybrid General Mode).

### General System Automations
These automations are highly recommended regardless of your energy provider:
*   [EV Free Window Reminder](./automations/ev_free_window_reminder.yaml) - Reminds you to plug in an EV before an 11:00 AM free power window.
*   [GoodWe Time Sync](./automations/goodwe_time_sync.yaml) - Keeps the inverter clock perfectly synced with Home Assistant to prevent schedule drift.
*   [GoodWe Battery Fault Alert](./automations/goodwe_battery_fault_alert.yaml) - Critical mobile alerts for BMS errors or warnings.

### Template Sensors
*   [GloBird Zero Hero Live Sensors](./sensors/) - Creates custom dashboard sensors to track your exact peak export amount and calculate your session profit in real-time.

---

## How to Use These Automations

1. Browse the `automations` folder for your specific energy provider.
2. Read the provider's strategy guide to choose the automation method that best suits your setup.
3. Open the chosen YAML file and copy the code.
4. In Home Assistant, navigate to **Settings > Automations & Scenes**, and create a new automation.
5. Click the three dots in the top right corner and select **Edit in YAML**.
6. Paste the code.
7. **CRITICAL STEP:** Use `Ctrl+F` to find the `# EDIT:` comments in the code. You **must** replace the placeholder entity IDs (like `select.inverter_operation_mode`) with the exact entity IDs used by your specific Home Assistant setup.
8. Save and monitor your first few cycles to ensure everything is functioning correctly.

---

**Disclaimer:** *These automations are provided as-is. Always test your automations while monitoring your inverter to ensure they behave as expected. We are not responsible for unexpected grid charges, battery drain, or hardware wear.*
