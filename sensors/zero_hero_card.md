# Zero Hero Dashboard Card

A Lovelace card for managing the Zero Hero helpers without opening the Helpers settings page.
Drop the YAML below into a dashboard in raw edit mode.

Two versions are provided:
- **Standard** - works with any HA install, no custom cards required
- **Mushroom** - nicer UI if you have the Mushroom Cards integration installed (via HACS)

---

## Standard version (no custom cards required)

```yaml
type: entities
title: Zero Hero Controls
icon: mdi:flash
entities:
  - entity: input_boolean.zero_hero_enabled
    name: Zero Hero Enabled

  - type: divider

  - type: section
    label: Tariff Rates

  - entity: input_number.zero_hero_rate_super
    name: Super Export Rate ($/kWh)
    icon: mdi:lightning-bolt

  - entity: input_number.zero_hero_rate_base
    name: Base Export Rate ($/kWh)
    icon: mdi:lightning-bolt-outline

  - entity: input_number.zero_hero_rate_credit
    name: Daily Zero-Grid Credit ($)
    icon: mdi:currency-usd

  - entity: input_number.zero_hero_super_cap
    name: Super Rate Cap (kWh)
    icon: mdi:gauge

  - type: divider

  - type: section
    label: Session (read-only)

  - entity: sensor.zero_hero_session_export
    name: Tonight's Export
    icon: mdi:transmission-tower-export

  - entity: sensor.zero_hero_session_profit
    name: Tonight's Profit
    icon: mdi:cash-plus
```

---

## Mushroom version (requires Mushroom Cards via HACS)

```yaml
type: vertical-stack
cards:
  - type: custom:mushroom-title-card
    title: Zero Hero
    subtitle: GloBird peak export controls

  - type: custom:mushroom-entity-card
    entity: input_boolean.zero_hero_enabled
    name: Enabled
    icon: mdi:flash
    tap_action:
      action: toggle
    layout: horizontal

  - type: custom:mushroom-title-card
    title: ""
    subtitle: Tariff rates

  - type: custom:mushroom-number-card
    entity: input_number.zero_hero_rate_super
    name: Super export rate
    icon: mdi:lightning-bolt
    display_mode: buttons
    layout: horizontal

  - type: custom:mushroom-number-card
    entity: input_number.zero_hero_rate_base
    name: Base export rate
    icon: mdi:lightning-bolt-outline
    display_mode: buttons
    layout: horizontal

  - type: custom:mushroom-number-card
    entity: input_number.zero_hero_rate_credit
    name: Daily zero-grid credit
    icon: mdi:currency-usd
    display_mode: buttons
    layout: horizontal

  - type: custom:mushroom-number-card
    entity: input_number.zero_hero_super_cap
    name: Super rate cap (kWh)
    icon: mdi:gauge
    display_mode: buttons
    layout: horizontal

  - type: custom:mushroom-title-card
    title: ""
    subtitle: Tonight's session

  - type: custom:mushroom-entity-card
    entity: sensor.zero_hero_session_export
    name: Exported
    icon: mdi:transmission-tower-export
    layout: horizontal

  - type: custom:mushroom-entity-card
    entity: sensor.zero_hero_session_profit
    name: Profit
    icon: mdi:cash-plus
    layout: horizontal
```

---

## Notes

- The session sensors (`zero_hero_session_export` and `zero_hero_session_profit`) only show values during an active peak window. They read `0` outside of peak - this is correct.
- `zero_hero_export_start` is intentionally excluded from the card. It's managed automatically by the automation and editing it manually will throw off the profit calculation.
- If you only want the rate editors and don't have the template sensors set up, just remove the "Session" section.
