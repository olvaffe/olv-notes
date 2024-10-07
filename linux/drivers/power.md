Kernel Power
============

## Powr Supply

- `power_supply_register` registers a `power_supply`
  - `power_supply_desc` describes the power supply
    - `power_supply_type`
      - `POWER_SUPPLY_TYPE_BATTERY` is battery
      - `POWER_SUPPLY_TYPE_MAINS` is ac
      - more
    - `power_supply_property`
      - `POWER_SUPPLY_PROP_STATUS` is 1 if battery is charging
      - `POWER_SUPPLY_PROP_ONLINE` is 1 if ac is connected
      - more
- `power_supply_changed` notifies state change
- note that it is not just for the system
  - a battery-powered hid device can register a power supply as well, to
    report the battery capacity
