Kernel hwmon
============

## Core

- `devm_hwmon_device_register_with_info` registers an `hwmon_chip_info`
  - other subsys can register an hwmon chip automatically as well
    - power supply core calls `power_supply_add_hwmon_sysfs`
    - thermal core calls `thermal_add_hwmon_sysfs`
    - nvme calls `nvme_hwmon_init`
    - ufs calls `ufs_hwmon_probe`
- userspace
  - lm-sensors, <https://github.com/lm-sensors/lm-sensors>
  - `libsensors` parses `/sys/class/hwmon`
