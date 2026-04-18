# GloBird Zero Hero Automations & Strategy Guide

This folder contains automations designed to manage GoodWe ESA inverters on the GloBird Zero Hero energy plan (or similar dynamic Time-of-Use VPP plans). 

The YAML files are located in the subfolders. The section below explains the logic behind the automations and the differences between the available methods of controlling GoodWe inverters.

## Automating the GloBird Zero Hero Plan

The Zero Hero plan has two distinct windows that require different inverter behaviours to maximise profit and battery health.

### 1. The Free Window (11:00 AM to 2:00 PM)
Power is free during this period. The automation forcefully charges the battery to 100% using grid power if solar generation is insufficient. 

The GoodWe ESA series uses Lithium Iron Phosphate (LFP) battery chemistry. Unlike older lithium-ion batteries, it is not harmful to charge LFP batteries to 100%, provided they do not sit at full capacity for extended periods. Regularly bringing the battery to 100% is actually required to calibrate the Battery Management System (BMS). Charging during this window ensures there is enough capacity to get through the night, cover the next morning's load, and leave enough excess capacity to export for profit during the evening.

### 2. The Peak Window (6:00 PM to 8:00 PM / 9:00 PM)
Grid power is expensive to import during this window, but GloBird offers a high "Super Export" rate (e.g., 15c/kWh for the first 10kWh or 15kWh) and a daily "Zero-Grid" credit if nothing is imported. 

The automation goal during this window is to maximise profit while minimising wear and tear on the battery. Rather than dumping the entire battery capacity into the grid, the objective is to export the minimum amount necessary to hit the 10kWh or 15kWh Super Export cap (securing the $1.50 or $2.25 credit) while continuing to cover the standard house load. While users can choose to export beyond the cap for the baseline rate (usually ~6c/kWh), it is generally not worth the resulting degradation on the battery lifespan.

## Understanding GoodWe Memory Types

Before configuring an automation for a GoodWe ESA, it is important to understand how the inverter stores its settings. 

When the standard Home Assistant integration is used to change the inverter's operation mode (such as switching from General Mode to Eco Mode) or to change Time of Use (TOU) charging limits, those commands are written directly into the inverter's EEPROM (Flash Memory). 

Writing to the Flash Memory has two impacts:
1. It immediately overwrites and erases any Time of Use (TOU) schedules configured in the GoodWe SEMS+ mobile app.
2. It consumes write cycles. Flash memory has a physical hardware limit (often rated around 100,000 cycles). 

Because automating these changes multiple times a day introduces unnecessary hardware wear and software conflicts, there are several distinct approaches to handling the evening peak export. 

## Methods for Managing Evening Peak Export

### [The Baseline: SEMS+ App Only (No HA Automation)]
For users wanting the simplest and safest setup, you can skip Home Assistant automations entirely. You simply use the GoodWe SEMS+ app to programme a TOU schedule that charges the battery at 11:00 AM and discharges it at 6:00 PM. Home Assistant is used purely to monitor and graph the data.
* **The Downside:** SEMS+ is highly limiting. It only allows you to restrict the total inverter power, not the specific grid export rate, making it very imprecise. Furthermore, it is a static schedule that cannot react to fluctuating household loads or low battery levels (though a bug patch for SOC limitations is pending from GoodWe) above the raw inverter output percentage set. 

### [Method 1: Standard Eco Mode](./method1_standard/)
This method uses the standard Home Assistant integration to switch the inverter into "Eco Mode" at 6:00 PM. 
* **How it works:** When Home Assistant triggers Eco Mode, it actually writes your discharge command into the very first Time of Use (TOU) slot in the inverter's register. 
* **The Downside:** Because it commandeers that first TOU slot, it instantly erases any existing schedules you had set up in the SEMS+ app. This is why many users find this method "doesn't work" for them—it breaks their other schedules. Additionally, switching modes and altering export parameters writes to the inverter's flash memory multiple times a day.

### [Method 2: Power User EMS (RAM Commands)](./method2_ems/)
This approach bypasses flash memory entirely by using a third-party, experimental integration (available via HACS) to send raw Energy Management System (EMS) commands directly to the inverter's temporary RAM. 
* **How it works:** The integration provides two specific entities that function together: an `EMS Mode` dropdown and an `EMS Power Limit` number box (measured in Watts). When the automation sets the mode to `Export AC` and the power limit to `5000`, the inverter automatically prioritises the house load first, and then pushes exactly 5000W of excess power directly to the grid. 
* **The Downside:** Because it relies on an unofficial, community-built integration, a future GoodWe firmware update could break the EMS registers. It is also reliant on a stable Home Assistant connection; if your server crashes while an EMS command is active, the inverter may get stuck in that state (though this can be easily rectified by manually restarting the inverter).

### [Method 3: Hybrid General Mode (Recommended)](./method3_hybrid/)
This method utilises both the SEMS+ app and Home Assistant to create a safe, dynamic system that avoids flash memory wear. It covers the limitations of SEMS+ by using Home Assistant to provide intelligent logic.

* **How it works:** You only need to programme the evening *discharge* TOU schedule into the SEMS+ app (setting it to discharge at 100% during the 6:00 PM peak). In Home Assistant, the inverter is left in General Mode. 

When the 6:00 PM peak begins, the SEMS+ rule tells the battery to aggressively discharge. However, the HA automation acts as the "brains", enforcing a strict grid export limit (e.g., 5000W). This safely prioritises the house load while ensuring the excess export reliably hits, but never exceeds, the targeted cap. 

Additionally, this HA automation manages the force-charging during the 11:00 AM free window natively (meaning you don't need a charge schedule in SEMS+ at all), and dynamically drops the evening export limit to 0W if it detects the battery State of Charge (SOC) is too low to survive the night, protecting you from expensive grid imports. 

## Required Home Assistant Helpers

If you are using Methods 1, 2, or 3, the automations utilise two Home Assistant "Helpers" to track peak export data and calculate session profit. These must be created prior to running the YAML files.

**1. The Enable/Disable Toggle**
This allows the automation to be quickly paused (e.g., when away on holiday or when running heavy loads requiring battery conservation).
* Go to **Settings > Devices & Services > Helpers**.
* Click **+ CREATE HELPER** and select **Toggle**.
* Name it: `Zero Hero Enabled`
* Entity ID generated: `input_boolean.zero_hero_enabled`

**2. The Export Math Helper**
This temporarily stores the total export metric at the start of the 6:00 PM peak, allowing the automation to calculate the exact session profit at the end of the window.
* Go to **Settings > Devices & Services > Helpers**.
* Click **+ CREATE HELPER** and select **Number**.
* Name it: `Zero Hero Export Start`
* Set **Minimum value** to `0`, **Maximum value** to `100000`, and **Step size** to `0.1`.
* Entity ID generated: `input_number.zero_hero_export_start`
