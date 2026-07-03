# Tesla Charge Orchestration - Optional Add-on

An optional layer on top of the GloBird Zero Hero setup for households that charge a Tesla at home. It aims the car at the free window, keeps it out of the peak window, tops it up from the house battery overnight when there's genuinely spare energy, and tells you what the whole arrangement is worth in dollars and petrol-equivalent terms.

**This is an add-on, not a requirement.** Everything in the rest of this repo works without it. Install it if you have a Tesla, charge at home, and want the charging decisions made for you.

## What you need

- **[Teslemetry](https://teslemetry.com/)** (HACS or core integration, paid subscription) - this is the hard requirement. Teslemetry exposes the car's SOC, charge switch, amp control, and charge-limit control to HA. The Tesla Wall Connector's own integration is **read-only** (it can tell you a car is plugged in and how much energy has flowed; it cannot start, stop, or throttle anything), so without Teslemetry there is nothing for these automations to actuate.
- A working Zero Hero setup from this repo (any HA method). The peak blocker keys off `input_boolean.zero_hero_enabled`, and the savings sensors read the `globird_*` rate helpers from the [cost-tracking sensors](../../sensors/globird_cost_tracking.yaml).
- Optionally, the Tesla Wall Connector integration for the energy meter (feeds the savings tracker) and guest-car detection.

> **Entity naming:** Teslemetry names entities after your car. These files use `yourcar` as the placeholder (`sensor.yourcar_battery_level`, `switch.yourcar_charge`, ...) - replace it everywhere with your car's actual slug (Developer Tools > States, search your car's name). Every reference carries an `# EDIT:` marker as usual.

## The strategy in one paragraph

The car charges **free** 11:00-14:00, as hard as the site can bear without sabotaging the house battery's own free-window charge. From 14:00 it may **trickle** gently from the house battery - but only while the car is below its trickle target AND the house battery stays above every reserve layer. From 16:00 to 23:00 (the import-peak window) charging is **blocked** outright, with a one-tap override for the days you genuinely need the range. After 23:00 the trickle logic re-evaluates. The car's own charge limit does the final enforcement, so a lost HA connection never overcharges anything.

## Design principles (why it's built the way it is)

- **Fail-safe defaults.** Every decision that can't be made safely resolves to "don't charge" or "stop charging". The house battery's reserve is checked against *two* floors (a helper you set, plus a computed overnight-need backstop) before a single trickle amp flows.
- **Master-switch staging.** All three car-commanding automations gate on `input_boolean.tesla_orchestrator_enabled`, which you create **off**. Install everything, watch a full day of traces, then flip it on. Rollback is the same switch. The notification-only automations ship ungated - they command nothing, so there's no behaviour to stage.
- **API-call minimisation.** Every Tesla command is wrapped in "only send if the state is actually different". Tesla's API is rate-limited and each call wakes electronics in the car; a well-behaved orchestrator is mostly silent. Enforcement is delegated car-native where possible - the charge *limit* is set once and the car polices it, rather than HA babysitting the SOC.
- **The ESA automation is untouched.** Nothing in this folder writes to the inverter or the export limit. The house battery automation and the car orchestrator communicate only by reading the same sensors.

## What's in this folder

| File | What it does | Master-gated? |
|---|---|---|
| [`tesla_charge_orchestrator.yaml`](./tesla_charge_orchestrator.yaml) | The brain: free-window dynamic amps, 14:00 handover, 16:00 stop, 23:00 trickle evaluation, live cutoffs | Yes |
| [`tesla_peak_charge_blocker.yaml`](./tesla_peak_charge_blocker.yaml) | Stops any charging that starts during 16:00-23:00; actionable notification with one-tap override | Yes |
| [`tesla_override_handler.yaml`](./tesla_override_handler.yaml) | Handles the "Charge Anyway" tap: sets the override boolean and restarts charging | Yes |
| [`tesla_free_window_reminder.yaml`](./tesla_free_window_reminder.yaml) | 10:45 "plug in now" nudge - only when the car is home, unplugged, and actually low | No (notify-only) |
| [`tesla_window_summary.yaml`](./tesla_window_summary.yaml) | 14:05 result: kWh added, ~km gained, $ saved vs shoulder rate | No (notify-only) |
| [`tesla_night_closure_check.yaml`](./tesla_night_closure_check.yaml) | 22:00: doors/windows/frunk/trunk/lock check, lists exactly what's open | No (notify-only) |
| [`tesla_guest_car_detection.yaml`](./tesla_guest_car_detection.yaml) | A car plugged into the Wall Connector that isn't yours | No (notify-only) |
| [`ev_savings_sensors.yaml`](./ev_savings_sensors.yaml) | Utility meters + template sensors: daily/monthly savings, weekly petrol-equivalent | n/a (sensors) |

If you were using the generic [`ev_free_window_reminder.yaml`](../ev_free_window_reminder.yaml) from the automations root, the Tesla reminder **replaces** it - same job, but SOC-aware (it stays quiet when the car doesn't need charge) and it confirms when you plug in. Disable the generic one.

## Required helpers

Thirteen helpers, all created via **Settings > Devices & services > Helpers** (same flow as the [strategy guide's helpers](../globird/README.md#required-home-assistant-helpers)). Starting values in brackets.

**Toggles** (both default off - leave them off):

| Helper | Entity ID | Purpose |
|---|---|---|
| Tesla Orchestrator Enabled | `input_boolean.tesla_orchestrator_enabled` | Master switch. **Create it OFF and leave it off until you've watched a full day of traces.** |
| Tesla Charge Override | `input_boolean.tesla_charge_override` | Suspends all charge control until 23:00 or unplug. Set by the one-tap notification action; you can also flip it manually before a road trip. Auto-clears. |

**Numbers** (name / entity ID / min-max-step / unit / starting value):

| Helper | Entity ID | Range | Start |
|---|---|---|---|
| Tesla Charge Limit Target | `input_number.tesla_charge_limit_target` | 50-100, step 5, % | `90` |
| Tesla Free Window Max Amps | `input_number.tesla_freewindow_max_amps` | 5-32, step 1, A | `32` |
| Tesla House Batt Window Target | `input_number.tesla_house_batt_window_target` | 30-100, step 5, % | `55` |
| Tesla Trickle SOC Target | `input_number.tesla_trickle_soc_target` | 20-90, step 5, % | `50` |
| Tesla Trickle Batt Floor | `input_number.tesla_trickle_batt_floor` | 40-80, step 5, % | `50` |
| Tesla Trickle Amps | `input_number.tesla_trickle_amps` | 5-16, step 1, A | `8` |
| Tesla Reminder SOC Threshold | `input_number.tesla_reminder_soc_threshold` | 20-100, step 5, % | `60` |
| Tesla Efficiency | `input_number.tesla_efficiency_kwh_100km` | 10-25, step 0.5, kWh/100km | `13` |
| Tesla Overnight House kWh | `input_number.tesla_overnight_house_kwh` | 5-30, step 0.5, kWh | `16` |
| ICE Fuel Consumption | `input_number.ice_fuel_consumption` | 3-15, step 0.5, L/100km | `5.0` |
| ULP Price Per Litre | `input_number.ulp_price_per_litre` | 1-3, step 0.01, AUD | `1.85` |

Notes on the interesting ones:

- **Charge limit target `90`:** on an LFP-pack Tesla (RWD models), Tesla's own guidance is that charging to 100% regularly is fine and a weekly 100% charge keeps the BMS calibrated - so if yours is LFP, feel free to run this at 100, or slide it up before a long trip. The 90 default is the conservative pick that suits both chemistries. (Long-range/performance NMC packs should live at 80-90.)
- **House Batt Window Target `55`:** the house-battery SOC the free window is expected to reach *even while the car is charging*. The orchestrator throttles the car, never the house battery - see the dynamic-amps note below.
- **Trickle floor vs overnight kWh:** the trickle draws from the house battery only while it's above `tesla_trickle_batt_floor` **and** above a computed backstop of `10% + (overnight house kWh as % of battery)`. The helper floor is your policy; the backstop is physics. Set `tesla_overnight_house_kwh` from your actual evening-to-11am consumption (check your energy dashboard over a week). If you have a learned/forecast overnight figure from another system, point the backstop template at that instead - it's a one-line edit flagged in the YAML.

## The dynamic amps formula (free window)

At 11:00 (and re-evaluated at 12:00 and 13:00), the orchestrator computes the highest charge current the site can give the car **without the house battery missing its own window target**:

1. Project where the house battery will be at 14:00 if the car takes its maximum amps (house battery gets whatever grid headroom remains, plus DC solar, capped at its own charge-rate ceiling).
2. If the projection clears `tesla_house_batt_window_target`: the car gets full amps.
3. If not: work out the grid power the battery *needs* to hit the target, give the car what's left, rounded **down** to a 32/24/16/8A tier (tiers avoid thrashing the car's charger with odd values on every re-eval).

The formula carries four site constants flagged `# EDIT:` in the YAML: your grid import cap (`14.1` kW default - single-phase QLD supply), kW-per-amp at your nominal voltage (`0.235` = 235V single-phase), your house battery's usable kWh (`48`) and its max charge rate (`10` kW). Get these right or the projection is fiction. DC-coupled solar counts toward the battery's budget because it doesn't compete with the car for grid headroom - that's the AC+DC blending story from the [strategy guide](../globird/README.md) working in your favour again.

## Install order

1. Create the 13 helpers. **Both toggles off.**
2. Add [`ev_savings_sensors.yaml`](./ev_savings_sensors.yaml) to your config (`utility_meter:` keys merge into any existing block; template sensors join your `template:` section). Enable the odometer entity if you want the km meter - Teslemetry ships it disabled (Settings > Devices & services > Teslemetry > entities > enable `sensor.yourcar_odometer`). **Full restart** (utility meters need it).
3. Paste the seven automations, replacing every `# EDIT:` (car slug, GoodWe sensors, notify services).
4. Watch one full day with the master switch off. The notification-only automations run immediately (they're harmless); the three car-commanding ones will show "condition not met" traces.
5. Flip `input_boolean.tesla_orchestrator_enabled` on. Watch the first live window. Rollback = flip it off.

## Known limits

- **The Wall Connector can't be commanded.** Gen 3 TWC is read-only over local API. Everything here actuates the *car*. A guest's EV on your charger is detect-and-notify only ([`tesla_guest_car_detection.yaml`](./tesla_guest_car_detection.yaml)), and its energy lands in your EV meters (known skew - the savings figures assume the kWh went into your car).
- **One car.** The logic assumes the Teslemetry car and the TWC car are the same vehicle. Two-EV households need a second set of helpers and automations, and a way to tell whose cable is whose - not covered here.
- **Times are Zero Hero's windows.** 11-14 free, 16-23 peak. On a different plan, sweep every time trigger and window condition (they're all flagged).
