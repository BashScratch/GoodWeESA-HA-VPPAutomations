# GoodWe ESA - Home Assistant VPP Automations

Welcome. This repository is a collection of Home Assistant YAML automations, guides, and template sensors designed specifically for GoodWe ESA series inverters.

If you are connected to a Virtual Power Plant (VPP) or a dynamic wholesale energy plan (like GloBird Zero Hero, Amber Electric, etc.), configuring your inverter to charge and discharge at the right times is critical.

---

## The GoodWe "Flash Memory" Dilemma

Before you automate a GoodWe inverter for a VPP, you need to understand how the inverter stores its settings. 

When you use Home Assistant to change the inverter to "Eco Mode" and set a discharge limit, it overwrites the Time of Use (TOU) schedule in the inverter's EEPROM (Flash Memory). Flash memory has a limited number of write cycles (typically ~100,000). Doing this twice a day will also wipe out any schedules you have set up in the GoodWe SEMS+ app.

Because of this, the Home Assistant community has developed three different ways to handle VPP peak exporting. You must choose the method that best fits your setup.

### Method 1: The "Send It" Approach (Standard Eco Mode)
* **How it works:** You use the official Home Assistant integration to toggle Eco Mode and set discharge limits daily. 
* **Pros:** Forces the battery to export during peak times for maximum profit. Very easy to set up.
* **Cons:** Wipes your SEMS+ TOU schedule. Writes to flash memory daily (though modern chips should easily survive 10+ years of this).

### Method 2: The Power User Approach (Experimental EMS)
* **How it works:** You install the [Experimental GoodWe HACS Integration](https://github.com/mletenay/home-assistant-goodwe-inverter) and use real-time `EMS Mode` commands (`Export AC`) to force discharge.
* **Pros:** Maximum profit. Zero flash memory wear. Does not erase your SEMS+ schedules.
* **Cons:** Requires installing third-party custom repositories via HACS. More complex to configure.

### Method 3: The Hybrid / Safe Approach (General Mode)
* **How it works:** You set up a static nighttime discharge schedule inside the GoodWe SEMS+ app. In Home Assistant, you leave the inverter in General Mode, and simply automate the `grid_export_limit` to cap how fast it is allowed to discharge.
* **Pros:** 100% safe. Zero flash wear. Protects battery longevity. Perfect for plans where you just want a "Zero-Grid" daily bonus.
* **Cons:** Won't aggressively force the battery to dump 5kW into the grid if the house load is low, meaning you may miss out on some high Feed-in Tariff (FiT) profits.

---

## Available Automations

*(Note: We recommend starting with Method 3 if you are unsure).*

*   [Method 1 & 2: GloBird Eco/EMS Command](./automations/globird_eco_method.yaml) *(Coming Soon)*
*   [Method 3: GloBird General Mode Command](./automations/globird_general_method.yaml) - The Hybrid/Safe method. Automates free grid charging (11am-2pm) and manages the peak export limit (18:00-21:00).
*   [EV Free Window Reminder](./automations/ev_free_window_reminder.yaml) - Reminds to plug in EV before the 11am free window.
*   [GoodWe Time Sync](./automations/goodwe_time_sync.yaml) - Keeps inverter clock synced with Home Assistant.
*   [GoodWe Battery Fault Alert](./automations/goodwe_battery_fault_alert.yaml) - Critical mobile alerts for BMS errors or warnings.
