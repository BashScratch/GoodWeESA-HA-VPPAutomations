# GoodWe VPP Template Sensors

This folder contains YAML code for Home Assistant Template Sensors. Unlike automations, template sensors cannot be set up via the visual UI. They must be added to your configuration files.

***

## How to Install the GloBird Zero Hero Sensors

These sensors calculate your live export (in kWh) and your total profit (in AUD) during the evening peak window. 

### Step 1: Check your configuration structure
Before pasting the code, check your `configuration.yaml` file (accessible via the File editor or Studio Code Server add-on). 
* Look to see if you have a line that says `template: !include templates.yaml`. 
* If you do, you will paste the code into your `templates.yaml` file. 
* If you do not, you will paste the code directly into `configuration.yaml` under a `template:` heading.

### Step 2: Paste the Code
Copy the YAML block from `globird_zero_hero_sensors.yaml`. 
* Ensure the indentation is correct. The `- sensor:` block should be indented two spaces if placing it directly under `template:` in the main configuration file.
* **CRITICAL:** Use `Ctrl+F` to find the `# EDIT:` comment and replace `sensor.goodwe_total_energy_export` with your exact GoodWe total export entity ID.

### Step 3: Restart Home Assistant
Go to **Developer Tools > YAML** and click **Check Configuration**. If it shows a green tick, click **Restart** (or **Reload Template Entities**) to make your new sensors go live.
