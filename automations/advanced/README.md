# Advanced automations (optional)

These automations are layered on top of one of the four main methods. They aren't required to make Zero Hero work; they're protective and quality-of-life additions. Pick the ones that match your setup.

Each YAML file in this folder is independent. You can add any subset, in any order. The headers explain the helpers and entities each one expects.

## What's here

### [`grid_voltage_soak.yaml`](./grid_voltage_soak.yaml)

Watches grid voltage. If it climbs above your trigger threshold (default 252V, since AS/NZS 4777 inverters trip at 253V), turns on a configurable dump load (heat pump hot water, EV charger, pool pump, anything resistive) to soak the surplus and pull voltage back down. Releases when voltage falls back to a hysteresis threshold (default 247V).

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

### [`../sensors/goodwe_polarity_fix.yaml`](../../sensors/goodwe_polarity_fix.yaml) (template sensor)

A correctness fix, not a feature. Some GoodWe ARM firmware revisions report active power with an inverted sign (export shows positive, import shows negative). If your energy dashboard or this guide's profit calculations look backwards, set the polarity-inverted helper to ON and use the corrected sensor.

**When to use:** your `sensor.goodwe_active_power` shows the wrong sign convention.

## Future additions parked

A few automations and sensors have been sketched in the project's plan but not yet implemented. They'll land in a future pass:

- **Zero-Grid Credit watchdog** ([`../globird/zero_grid_credit_watchdog.yaml`](../globird/zero_grid_credit_watchdog.yaml) - early implementation, needs the `utility_meter` helper set up before it works) - alerts you when grid import during the peak window threatens GloBird's daily Zero-Grid credit.
- **Dynamic export limit** - template sensor that adapts the export limit to live house demand instead of a static number.
- **EV charge ring-fencing** - binary sensor that gates EV charging on battery state and tariff window. Vendor-specific to each EV charger's API; documented in the plan but not implemented because it depends on your charger.
- **Dynamic EV load balancing** - template sensor calculating available grid headroom and pushing the value to your EV charger's API. Same vendor-dependence as ring-fencing.

## Common gotchas

- **Test with the master toggle off first.** Each automation has an `input_boolean.*_enabled` switch. Leave it off, watch the trace fire (Settings > Automations > click the automation > Traces tab) on a couple of triggers, then enable when you're satisfied.
- **The smart plug entities are placeholders.** Every `# EDIT:` marker assumes you have an existing switch entity for the fan, dump load, etc. If you don't, set those up first - any HA-compatible smart plug works.
- **Polling-rate considerations apply.** These automations read live sensors that come from the GoodWe Modbus integration. If your integration is polling too aggressively (faster than 15 seconds), see [prereq 02](../../prerequisites/02_install_ha_integrations.md) for the polling note.
