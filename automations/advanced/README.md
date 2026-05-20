# Advanced automations (optional)

These automations are layered on top of one of the four main methods. They aren't required to make Zero Hero work; they're protective and quality-of-life additions. Pick the ones that match your setup.

Each YAML file in this folder is independent. You can add any subset, in any order. The headers explain the helpers and entities each one expects.

## What's here

### [`grid_voltage_soak.yaml`](./grid_voltage_soak.yaml)

Watches grid voltage. If it climbs above your trigger threshold (default 252V - AS/NZS 4777.2 sets 253V as the steady-state maximum, with actual inverter trip thresholds at a 10-minute sustained-average limit around 255-258V depending on DNSP configuration, plus instantaneous trips around 260V (1-2 second) and 265V (0.2 second)), turns on a configurable dump load (heat pump hot water, EV charger, pool pump, anything resistive) to soak the surplus and pull voltage back down before it climbs further. Releases when voltage falls back to a hysteresis threshold (default 247V).

**When to use:** you're in a high-PV-density street where grid voltage gets pushed up on hot afternoons, and you've seen your inverter curtail or trip off as a result.

**Requires:** a switch entity that controls the dump load (smart plug, ESPHome relay, Tuya/Zigbee switch, whatever).

### [`inverter_thermal_management.yaml`](./inverter_thermal_management.yaml)

Watches the inverter's internal temperature sensor. If it climbs above your trigger threshold (default 55°C), turns on a configurable cooling fan via a smart plug. Releases at hysteresis threshold (default 48°C).

**When to use:** your inverter is in a thermally challenging location (north-facing wall, hot garage, small enclosure) and you've seen it thermally derate during sustained high-export days.

**Requires:** a switch entity for an external fan, plus the inverter's internal temperature sensor (`sensor.goodwe_inverter_temperature` or your equivalent).

### [`lfp_calibration_charge.yaml`](./lfp_calibration_charge.yaml)

Tracks how long it's been since the battery last hit 100% SOC. If 14 days pass without a full charge, fires a notification suggesting a manual top-up. LFP cells need periodic full charges to keep the BMS coulomb counter calibrated; without them, your SOC reading can drift by several percent.

**When to use:** you're not on Zero Hero (where daily 100% charges happen anyway), or you've been away with the system on hold and want to know when it's time to recalibrate.

**Requires:** an `input_datetime.last_full_charge` helper and an `input_number.lfp_calibration_days` helper (default 14).

### [`grid_voltage_sag_alert.yaml`](./grid_voltage_sag_alert.yaml)

The mirror image of `grid_voltage_soak.yaml`. Watches grid voltage and fires a notification when it sags below your alert threshold (default 216V - AS 60038 lower steady-state limit for a 230V nominal supply). Releases at a hysteresis threshold (default 220V) once voltage has been sustained above it for 5 minutes. There's no inverter-side protective action for low voltage (it's a network problem upstream), but you can record it and report it to your distributor if it becomes a pattern.

**When to use:** you suspect your local supply is weak (brownouts during high-demand evenings, lights dimming when the heat pump kicks in, repeated low-voltage warnings from the inverter), and you want a record.

**Requires:** the helpers `input_number.voltage_sag_alert_threshold`, `input_number.voltage_sag_release`, and `input_boolean.voltage_sag_enabled`. Pairs with [`../../sensors/goodwe_voltage_statistics.yaml`](../../sensors/goodwe_voltage_statistics.yaml) for the historical graph.

### [`../../sensors/goodwe_voltage_statistics.yaml`](../../sensors/goodwe_voltage_statistics.yaml) (statistics sensors)

Five built-in `statistics` platform sensors that summarise grid voltage over rolling 1-hour and 24-hour windows: 1h min/max, 24h min/max/mean. Drop them on a mini-graph-card and you can see voltage trends at a glance. The historical record is the diagnostic data you'd hand to your distributor if you ever raise a supply-quality complaint.

**When to use:** alongside `grid_voltage_sag_alert.yaml` for the graph half of the picture, or standalone if you just want to know what your supply looks like over time.

**Requires:** nothing beyond the GoodWe integration and a voltage sensor.

### [`../sensors/goodwe_polarity_fix.yaml`](../../sensors/goodwe_polarity_fix.yaml) (template sensor)

A correctness fix, not a feature. Some GoodWe ARM firmware revisions report active power with an inverted sign (export shows positive, import shows negative). If your energy dashboard or this guide's profit calculations look backwards, set the polarity-inverted helper to ON and use the corrected sensor.

**When to use:** your `sensor.goodwe_active_power` shows the wrong sign convention.

## Future additions parked

A few automations and sensors have been sketched in the project's plan but not yet implemented. They'll land in a future pass:

- **Zero-Grid Credit watchdog** ([`../globird/zero_grid_credit_watchdog.yaml`](../globird/zero_grid_credit_watchdog.yaml) - needs the `utility_meter` helper set up before it works; see [prereq 04 Step 6](../../prerequisites/04_create_helpers.md#step-6---create-the-utility-meter-helper-only-if-installing-the-zero-grid-credit-watchdog) for the walkthrough) - alerts you when grid import during the peak window threatens GloBird's daily Zero-Grid credit.
- **Dynamic export limit** - template sensor that adapts the export limit to live house demand instead of a static number.
- **EV charge ring-fencing** - binary sensor that gates EV charging on battery state and tariff window. Vendor-specific to each EV charger's API; documented in the plan but not implemented because it depends on your charger.
- **Dynamic EV load balancing** - template sensor calculating available grid headroom and pushing the value to your EV charger's API. Same vendor-dependence as ring-fencing.

## Common gotchas

- **Test with the master toggle off first.** Each automation has an `input_boolean.*_enabled` switch. Leave it off, watch the trace fire on a couple of triggers (**Settings > Automations & Scenes**, click the automation to open the editor, then click the **Traces icon** in the top right of the editor next to the YAML menu), then enable when you're satisfied. Older HA versions had this as a "Traces" tab on the automation details page.
- **The smart plug entities are placeholders.** Every `# EDIT:` marker assumes you have an existing switch entity for the fan, dump load, etc. If you don't, set those up first - any HA-compatible smart plug works.
- **Polling-rate considerations apply.** These automations read live sensors that come from the GoodWe Modbus integration. If your integration is polling too aggressively (HA's default is 10s; the integration's own docs recommend 30s minimum), see [prereq 02](../../prerequisites/02_install_ha_integrations.md) for the polling note.
