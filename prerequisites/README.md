# Prerequisites - start here if any of this is new

If you've just bought a GoodWe ESA and a GloBird Zero Hero plan and you're trying to make Home Assistant do clever things with both - welcome. This folder is the bit that has to happen before you can paste any of the YAML in this repo and have it work.

We assume you've got a Home Assistant install running somewhere (a Green, a Yellow, a Pi, an old laptop, whatever) and you can log into the web UI. Everything else, we'll walk through.

## What you need to do, in order

The guides are numbered. If you do them in order you won't get stuck. If you skip ahead you probably will, because each step assumes the previous one is done.

1. [Enable Modbus TCP on the inverter](./01_enable_modbus_on_inverter.md) - turns on the network port HA needs to talk to your GoodWe.
2. [Install the GoodWe integrations in Home Assistant](./02_install_ha_integrations.md) - the native one (built in) for Methods 1 and 3, plus the experimental HACS one if you're going with Method 3.
3. [Install the HA Companion app for notifications](./03_install_companion_app.md) - this is what fires the "Zero Hero Armed" / "Complete" alerts to your phone.
4. [Create the required helpers](./04_create_helpers.md) - the toggles and number inputs the automations read from. The full helper list lives in the strategy guide; this guide just shows you the click path.
5. [Find your actual entity IDs in Developer Tools](./05_find_your_entities.md) - every `# EDIT:` comment in the YAML needs you to substitute a real entity ID from your specific install. This is where you find them.
6. [How to paste a YAML automation into Home Assistant](./06_paste_yaml_automation.md) - the bit nobody warns you about. The HA UI hides the YAML editor by default and you have to actively unhide it.
7. [How to add template sensors to configuration.yaml](./07_add_template_sensors.md) - needed for the optional dashboard sensors. Different from automations; can't be done from the UI.
8. [Set up the SEMS+ TOU schedule (Method 4 only)](./08_sems_tou_schedule.md) - the half of Method 4 that lives on the inverter side, not the HA side.
9. [First-run checklist](./09_first_run_checklist.md) - what to watch on day one and how to bail out cleanly if something's wrong.

## What we don't cover

- Installing Home Assistant itself. The [official docs](https://www.home-assistant.io/installation/) are good and the right starting point depends on your hardware.
- Networking basics - finding your router's admin page, setting a static DHCP lease, that sort of thing. Most modern routers have decent UI for this; if yours doesn't, the manual is your friend.
- Solar industry jargon. The [glossary](../GLOSSARY.md) covers the GoodWe / GloBird / HA terminology specifically; broader solar terms are not our problem.

## Thanks to the Whirlpool community

A lot of what's in this folder is built on threads at [forums.whirlpool.net.au](https://forums.whirlpool.net.au) - Australian solar owners working out the GoodWe ESA in public, often before official documentation existed. We cite specific threads where they helped most. If you've got an unanswered question, the GoodWe and home automation forums there are the best place we've found to ask.

Notable threads:

- [Home Assistant setup with GoodWe inverter (9xv6wp84)](https://forums.whirlpool.net.au/thread/9xv6wp84) - the big general thread, including Modbus enablement.
- [GoodWe ESA - Setting export TOU with SOC limit (9n111qlk)](https://forums.whirlpool.net.au/thread/9n111qlk) - documents the gotcha that switching out of Eco mode deletes your TOU schedule.
- [GoodWe ESA maximum charge rate? (9kppp8k2)](https://forums.whirlpool.net.au/thread/9kppp8k2) - explains the 10kW AC + 3.5kW DC blending that gives you 13.5kW total via TOU schedule.

## A note on inverter firmware

GoodWe ships firmware updates from time to time and the menu wording inside the SEMS+ and SolarGo apps shifts between versions. Where we describe a menu path that might have moved, we say so. If a screen we describe has been renamed, look around for similar wording - the underlying setting almost always still exists, it's just been moved to a different sub-menu.
