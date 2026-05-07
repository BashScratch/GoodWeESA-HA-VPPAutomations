# 1. Enable Modbus TCP on your GoodWe ESA

## Why we're doing this

Home Assistant talks to the GoodWe ESA over your home network using a protocol called **Modbus TCP**. The inverter needs to be configured to accept those connections - without this, HA can poll the GoodWe Cloud (slow, sometimes broken) but cannot read or write directly to the inverter (which is what we need for everything in this repo).

The good news: it's a one-off setup. Once it's on, it stays on.

## Step 1 - Find the inverter's IP address

Your inverter is on your home Wi-Fi (or LAN, if you've cabled it). You need its IP address to talk to it.

Three ways to find it:

1. **Router admin page** (most reliable). Log into your router (usually `http://192.168.1.1` or `http://192.168.0.1`). Look for "Connected devices", "DHCP clients", "LAN clients" or similar. The inverter usually shows up as `Solar-WiFi-xxxx` or `GoodWe-xxxx`. Note the IP.
2. **SolarGo app** (if you've already used it during install). Open the app, go to the inverter's detail page, look for "IP" or "Network" - it's usually buried in the settings somewhere.
3. **Network scanner**. If you've got a tool like Fing, Angry IP Scanner, or `nmap` on your laptop, scan your subnet and look for a device on port 502 (Modbus) or with a hostname containing "GoodWe" or "Solar".

Write the IP down. You'll use it in the next steps and in HA later.

## Step 2 - Pin it to a static address

DHCP leases expire. If your inverter changes IP, HA loses the connection and your automations stop working silently.

In your router admin page, find the **DHCP reservation** or **static lease** section, and bind the inverter's MAC address to its current IP. Wording differs by router brand - look for "DHCP reservation", "Address Reservation", "Static IP", "Bind". If your router doesn't support this, set the inverter to a static IP from inside the SolarGo app's network settings instead.

## Step 3 - Confirm Modbus TCP is off (it almost certainly is)

**Modbus TCP is off by default on GoodWe ESA inverters.** This is the single most common reason people get to "install the GoodWe integration in HA" and it just sits there saying "Failed to connect" - the integration is fine, the inverter simply isn't accepting the connection because the port isn't open.

You'll almost certainly need to turn it on. Test first to confirm - on the rare chance an installer enabled it for you, you can skip ahead.

From a terminal on any computer on the same network as the inverter:

```bash
nc -vz <inverter_ip> 502
```

If you get `Connection succeeded` (or similar), Modbus TCP is already on - lucky you. Skip ahead to Step 6.

If you get `Connection refused` or `timeout` (the expected outcome for most people), continue to Step 4 to turn it on.

(If you don't have `nc`/`netcat` handy: a port-scanner like Fing or [PortQry](https://www.microsoft.com/en-us/download/details.aspx?id=17148) does the same job. The mletenay integration also fails to add the device with a meaningful "connection refused" error if Modbus is off, so installing the integration first and watching it fail is also a valid test.)

## Step 4 - Turn Modbus TCP on (if it isn't)

This is the bit where the official documentation goes quiet, and the Whirlpool community fills in the gap.

> **Heads up on app choice.** GoodWe is migrating consumer features from the older **SolarGo** app to the newer **SEMS+** app. For most day-to-day stuff (TOU schedules, dashboards) we now recommend SEMS+. **The Modbus TCP toggle, however, is in SolarGo** as of when this guide was written - or at least, that's where Mitch had to enable it. It may have moved to SEMS+ since. Check SEMS+ first if you're more comfortable there; if you can't find it, fall back to SolarGo.

Open the **SolarGo** app on your phone, log in, select your inverter, and look in the **settings / advanced settings / communication settings** menu for a Modbus TCP toggle. Wording varies between app versions:

- Some versions list it as **"Modbus TCP"** under Communication Settings.
- Some require entering a special code or installer password to access the menu.
- On some inverter firmware revisions the toggle isn't exposed in SolarGo at all and you need the **SEMS+ Pro** installer portal, or to ask your installer to enable it remotely.

If the option isn't where this guide says it is, [Whirlpool 9xv6wp84](https://forums.whirlpool.net.au/thread/9xv6wp84) has the most current community walkthrough including screenshots for several SolarGo versions and workarounds for installer-locked inverters. Search the thread for "Modbus" - it's been discussed at length.

> **If you're stuck behind an installer lock:** GoodWe-accredited installers can enable this remotely on your behalf. Most will do it on request - contact your installer with your inverter serial number and ask them to "enable Modbus TCP on port 502". They won't usually charge for it.

> **Installer password gotcha (SolarGo vs SEMS+):** if you have an installer-level login set up in SolarGo (either the default installer code or a custom password your installer or you set), **that password does not work in SEMS+**. The two apps don't share installer credentials. If you've recently switched to SEMS+ for installer-level access and your SolarGo password is being rejected, you need a separate installer password for SEMS+ - contact your installer for it, or set one up via SEMS+ directly if your account has the right level of access.

## Step 5 - Test again

Re-run the `nc` test from Step 3. If it now succeeds, you're sorted.

```bash
nc -vz <inverter_ip> 502
```

## Step 6 - Note your firmware version

While you're in the SolarGo app, write down the inverter firmware version (usually shown on the inverter detail page, sometimes labelled "ARM" and "DSP" versions). The mletenay integration occasionally has firmware-specific bugs and knowing your version makes troubleshooting much faster.

## What you should have at the end of this guide

- Your inverter's local IP address, written down somewhere.
- A static DHCP lease so the IP doesn't drift.
- Confirmed Modbus TCP is reachable on port 502.
- A note of your inverter firmware version.

If all four boxes tick, you're ready for the next guide.

## If it didn't work

- **Port 502 still refused after enabling Modbus in SolarGo.** Reboot the inverter (DC isolator off, wait 30 seconds, back on). Some firmware versions need a power cycle to apply the change.
- **You can't find Modbus TCP in SolarGo.** Your firmware version may not expose it. Update the inverter firmware via SolarGo (or have your installer do it), then look again.
- **Pinging works but Modbus doesn't.** Some routers run a firewall that blocks "unusual" ports between Wi-Fi and LAN. Check your router's firewall rules.
- **You're getting different IPs each day.** Static lease isn't sticking - try setting a static IP on the inverter side (SolarGo app's network settings) instead of relying on the router.
