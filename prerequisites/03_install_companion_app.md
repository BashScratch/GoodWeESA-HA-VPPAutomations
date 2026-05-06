# 3. Install the HA Companion app for notifications

Every automation in this repo can fire notifications to your phone - "Zero Hero Armed", "Battery Fault", and so on. Those notifications go via the **Home Assistant Companion app**, which is the official mobile app for HA.

You don't strictly need notifications for the automations to work - they'll still control the inverter without them. But you'll be flying blind during the first few days of running them, which is exactly when you most want to see what they're doing. We strongly recommend setting this up before enabling any of the YAML.

## Step 1 - Install the app

- **iOS:** [App Store - Home Assistant](https://apps.apple.com/au/app/home-assistant/id1099568401)
- **Android:** [Google Play - Home Assistant](https://play.google.com/store/apps/details?id=io.homeassistant.companion.android)

Both are free and published by Nabu Casa, the company that maintains Home Assistant.

## Step 2 - Connect the app to your HA instance

Open the app. It'll ask you to connect to a Home Assistant server.

If you have **Nabu Casa Cloud** (a paid HA subscription), the app can find your instance automatically. Just log in.

If you **don't** have Nabu Casa, you'll need:

- The local URL of your HA install (something like `http://homeassistant.local:8123` or `http://192.168.1.42:8123`).
- Your HA username and password.

Type the URL in, log in, and accept any prompts about background location, notifications, etc. Notifications must be allowed - that's the whole point.

## Step 3 - Find your notify service name

Once the app is connected, HA automatically creates a **notify service** for your phone. The service name is `notify.mobile_app_<your_device_name>`, where `<your_device_name>` is whatever HA decided to call your phone.

To find the exact name:

1. In HA's web UI, go to **Developer Tools > Actions** (older HA versions called this **Services**).
2. In the action dropdown, type `notify.mobile_app` - you should see one or more entries appear.
3. The full string after `notify.` is your service name. Note it down.

Common examples:

- `notify.mobile_app_iphone` - iPhone with default name
- `notify.mobile_app_mitch_iphone` - iPhone named "Mitch's iPhone"
- `notify.mobile_app_pixel_8` - Android Pixel 8

If you've connected multiple phones, each gets its own service. You can pick the one you want to be the alert phone, or set up a notify group later that fires to all of them.

## Step 4 - Test it

In **Developer Tools > Actions**:

1. Pick your `notify.mobile_app_*` action from the dropdown.
2. In the YAML/data field, paste:
   ```yaml
   message: "Test notification from HA"
   title: "It works"
   ```
3. Click **Perform action**.

Your phone should buzz within a couple of seconds. If it does, you're done.

If it doesn't:

- Check the app on your phone is logged in and showing data.
- Check your phone's OS-level notification permissions for the HA app.
- Battery-saver / Doze mode on Android can suppress notifications. Whitelist the HA Companion app.

## Step 5 - Replace `notify.mobile_app_your_device_name` in the YAML

Every automation in this repo uses `notify.mobile_app_your_device_name` as a placeholder - that's the bit you replace with your actual notify service name from Step 3.

You'll do the actual replacement when you paste each YAML, covered in [Guide 06](./06_paste_yaml_automation.md). Right now just write your real service name down where you can find it later.

## Critical notifications (battery fault alert)

The [`goodwe_battery_fault_alert.yaml`](../automations/goodwe_battery_fault_alert.yaml) automation uses HA's **critical notification** feature, which bypasses Do Not Disturb and rings even on silent. This is intentional - you want to know immediately if your battery is throwing errors.

This works on:

- **iOS:** out of the box, via Apple's Critical Alerts API. The first critical alert will prompt you to allow this in iOS settings - say yes.
- **Android:** behaviour depends on the OEM. Most modern Android phones honour the critical flag if you've granted full notification access to the HA app. Samsung and Xiaomi sometimes need extra battery-optimisation tweaks.

If you don't want the loud-bypass-DND behaviour, edit the YAML for that one automation and remove the `interruption-level: critical` and `critical: 1` lines. The notification will still fire, just at normal priority.

## What you should have at the end of this guide

- HA Companion app installed and logged in to your HA instance on at least one phone.
- A test notification successfully delivered from Developer Tools > Actions.
- Your real `notify.mobile_app_*` service name written down.

Next: [Create the required helpers](./04_create_helpers.md).
