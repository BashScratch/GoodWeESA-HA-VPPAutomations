# GoodWe ESA - Home Assistant VPP Automations ⚡

Welcome! This repository is a community-driven collection of Home Assistant YAML automations, guides, and template sensors designed specifically for **GoodWe ESA series inverters**.

If you are connected to a Virtual Power Plant (VPP) or a dynamic wholesale energy plan (like GloBird Zero Hero, Amber Electric, etc.), configuring your inverter to charge and discharge at the right times is critical. These ready-to-use YAML files aim to make it as easy as copy, paste, and edit, so you can stop manually managing your battery and let Home Assistant maximize your savings automatically.

---

## 🛠️ Prerequisites

Before you begin, ensure you have the following:
1. **Home Assistant:** Up and running.
2. **GoodWe Integration:** A working integration pulling data from your GoodWe ESA inverter. (We recommend the standard Home Assistant GoodWe integration or the SolarGo app integration if you are using custom MQTT setups).
3. **Control Access:** Your integration must allow you to *write* data to the inverter (e.g., changing Operation Modes, setting Depth of Discharge, and altering Eco Mode power).

---

## 📖 How to Use These Automations

1. Browse the `automations` folder in this repository.
2. Find the YAML file that matches your energy plan or desired scenario.
3. Open the file and copy the YAML code.
4. Open your Home Assistant instance, navigate to **Settings > Automations & Scenes**, and create a new automation.
5. Click the three dots in the top right corner and select **Edit in YAML**.
6. Paste the code.
7. **CRITICAL STEP:** Use `Ctrl+F` to find the `# EDIT:` comments in the code. You **must** replace the placeholder entity IDs (like `select.inverter_operation_mode`) with the exact entity IDs used by your specific Home Assistant setup.
8. Save and test!

---

## 📂 Available Guides & Automations

*   [GloBird Zero Hero Schedule](./automations/globird_zero_hero.yaml) - Automates free grid charging (11am-2pm) and peak exporting (6pm-9pm).
*   *(More coming soon!)*

---

**Disclaimer:** *These automations are provided as-is. Always test your automations while monitoring your inverter to ensure they behave as expected. We are not responsible for unexpected grid charges or battery drain.*
