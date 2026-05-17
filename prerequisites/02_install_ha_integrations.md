# 2. Install the GoodWe integrations in Home Assistant

There are two GoodWe integrations for Home Assistant. You need one or both depending on which method you're going with.

| You're using... | You need |
|---|---|
| Method 2 (Standard Eco Mode) | Native GoodWe integration only |
| Method 3 (EMS RAM Commands) | **Both** native + mletenay experimental (HACS) |
| Method 4 (Hybrid) - recommended | Native GoodWe integration only |

The native one is built into Home Assistant - no extras to install, just enable it. The experimental one comes from HACS and adds the EMS entities Method 3 needs.

## Part A - Native GoodWe integration (everyone needs this)

This is the one shipped with Home Assistant by default. No HACS, no extras.

1. In HA, go to **Settings > Devices & services**.
2. Click **Add integration** (bottom right).
3. Type **`GoodWe`** in the search bar and click the result.
4. Enter your inverter's IP address (from [Guide 01](./01_enable_modbus_on_inverter.md)).
5. Leave the protocol as default (UDP) unless you've specifically configured Modbus RTU.
6. Click **Submit**.

If it works, HA will discover the inverter and offer to add it to an Area. Pick whatever area makes sense (we usually use "Garage" or "Outside" depending on where the inverter physically is).

After adding, click into the integration and you should see a long list of sensors - battery SOC, PV power, grid import/export, mode, and so on. Don't worry about reading every one yet; we'll come back to specific entities in [Guide 05](./05_find_your_entities.md).

> **Important: set the polling interval to at least 30 seconds.** The GoodWe ESA's local Modbus implementation doesn't handle aggressive polling well. The HA GoodWe integration's default scan interval is **10 seconds** ([per the HA integration docs](https://www.home-assistant.io/integrations/goodwe/)), and the same page warns: *"frequent polling conflicts with updates to the Goodwe SEMS cloud portal... Reducing polling frequency to 30 seconds or 1 minute seems to help in such cases."* What that looks like in practice: the green COM LED on the inverter blinks to indicate a server error, the **SEMS+ portal stops updating**, and some HA sensors flick to "unavailable" because the inverter can't simultaneously serve HA's Modbus polls and the SEMS cloud channel.
>
> **Recommended values:**
>
> - **30 seconds** (HA's own recommendation). Safe default. Plenty fast for everything in this guide; the automations work on minute-or-greater timing.
> - **15 seconds** works for many users on a hardwired ethernet connection but isn't guaranteed - it sits between HA's recommended floor and the documented 10s problem zone. Only try this if you specifically want faster updates and are willing to back off if SEMS+ starts dropping.
> - **1 to 5 minutes** is the ultra-conservative setting if SEMS+ keeps dropping at 30s ([HA community thread converges around 5 minutes](https://community.home-assistant.io/t/goodwe-sems-polling-interval/565818), on the reasoning that the inverter only updates SEMS at 5-minute intervals anyway).
>
> **How to change the interval:** the path depends on your HA version. Older versions had a simple **three-dots menu > Configure > Scan interval** slider on the GoodWe integration. Current HA (per the [GoodWe integration docs](https://www.home-assistant.io/integrations/goodwe/)) instead has you do it in two steps:
>
> 1. **Settings > Devices & services**, click into your GoodWe integration entry.
> 2. Three-dots menu > **System options** > toggle **Enable polling for updates** off.
> 3. Create a custom automation that calls `homeassistant.update_entity` against your GoodWe entities on whatever schedule you want (a `time_pattern` trigger every 30 seconds is the simplest version).
>
> Either approach works. If your version still shows a "Configure" option with a scan-interval field, the simple slider path is fine - the automation route is for current versions where the field has moved. If SEMS+ is still dropping out even at 1-2 minute polling, the cause is more likely your inverter's connection type (Wi-Fi dongle vs hardwired ethernet) than the polling rate - see prereq 01.

## Part B - HACS (only needed for Method 3)

If you're going with **Method 4** (recommended) or **Method 2**, you can skip the rest of this guide.

HACS is the Home Assistant Community Store - a way to install integrations and frontend cards that aren't bundled with HA itself. It's safe and widely used, but it's a manual install.

The official HACS documentation is excellent and we won't try to write a better version of it. Follow:

**[https://hacs.xyz/docs/use/download/download/](https://hacs.xyz/docs/use/download/download/)**

The short version of what's involved:

1. Install the HACS files into your HA config directory. The official guide gives you the exact commands depending on whether you're running HA OS, Container, or Core.
2. Restart Home Assistant.
3. Go to **Settings > Devices & services > Add integration**, search for **HACS**, and add it.
4. HACS asks you to log in to GitHub once (it uses your GitHub account to fetch repository content). Follow the prompts.

When HACS is installed, you'll have a new "HACS" item in the HA sidebar.

## Part C - mletenay experimental GoodWe integration (only needed for Method 3)

This is the integration that adds EMS-mode and EMS-power-limit entities - the RAM-level controls Method 3 uses.

> **Heads up:** This integration is community-maintained and pokes registers the official integration doesn't expose. It can break with a future inverter firmware update. We recommend Method 4 partly to avoid this dependency. If you're picking Method 3 deliberately, proceed.

1. Open **HACS** in the HA sidebar.
2. Click the three-dots menu (top right) > **Custom repositories**.
3. Add the repository:
   - **Repository:** `https://github.com/mletenay/home-assistant-goodwe-inverter`
   - **Type:** Integration
   - Click **Add**.
4. Close that dialog. Search HACS for **"GoodWe Inverter"** and you should see the experimental version listed.
5. Click it, then click **Download**, accept the version, and let it install.
6. **Restart Home Assistant** when prompted (Settings > System > Restart).
7. After restart, go to **Settings > Devices & services > Add integration**, search for **GoodWe Inverter** (note: this is *different* from the built-in "GoodWe" - look for the one with "Inverter" in the name).
8. Add the integration with the same IP address you used for the native one.

You'll now have two GoodWe integrations running side by side, which is the supported configuration. The HACS one is built as a **layer over the native one** rather than a replacement: the native integration keeps providing the standard sensors and basic mode controls, and the HACS integration adds the entities the native one doesn't expose (Eco Mode power, EMS mode, fast-charging switch). Per mletenay's repo, if you ever uninstall HACS, the native integration takes over seamlessly and your existing entity history is preserved.

## Verify both integrations are working

Go to **Developer Tools > States** (we'll cover this properly in [Guide 05](./05_find_your_entities.md)) and search for `goodwe`. You should see:

- For native only: a list of sensors and one or two `select` entities.
- For native + experimental: the above, **plus** entities containing `ems` (only present with the experimental integration).

If you don't see anything: the integration didn't connect. Common causes are wrong IP address, Modbus TCP not enabled (back to [Guide 01](./01_enable_modbus_on_inverter.md)), or the inverter being asleep (it doesn't respond when there's no PV and no battery action - try again during daylight).

## What you should have at the end of this guide

- Native GoodWe integration installed and showing sensors.
- For Method 3 only: HACS installed, mletenay experimental GoodWe integration installed, EMS entities visible.

If both ticked, the next guide is [Companion app for notifications](./03_install_companion_app.md).

## If it didn't work

- **Native integration says "Failed to connect"** - IP wrong, or Modbus TCP off, or inverter asleep. Re-check Guide 01.
- **HACS download button greyed out** - your GitHub login expired. Sign out of HACS (three-dots menu) and back in.
- **Experimental integration installs but no `ems` entities appear** - your inverter firmware may not support EMS via the experimental driver. The integration's [GitHub issues](https://github.com/mletenay/home-assistant-goodwe-inverter/issues) are the place to ask.
- **Both integrations are fighting and entities have weird `_2` suffixes** - see [Guide 05](./05_find_your_entities.md) for how to identify which is which.
