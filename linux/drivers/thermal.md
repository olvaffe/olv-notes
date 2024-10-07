Kernel Thermal
==============

## Thermal Driver

- during init, the thermal driver
  - calls `devm_thermal_of_zone_register` to add thermal zones
  - calls `thermal_add_hwmon_sysfs` to add thermal zones to hwmon subsys
- the core calls `set_trips` to set trips
  - the driver must call `thermal_zone_device_update` when the temperature
    goes out-of-range

## Core

- when a thermal zone trips, `thermal_zone_device_update` is called
  - `handle_thermal_trip` handles the trips
  - `governor->manage` informs the governor, such as `step_wise_manage`
    - `thermal_zone_trip_update` updates `instance->target` for the new target
      state
      - if heating up, it picks a higher cooling state
      - if cooling down, it picks a lower cooling state
    - `thermal_cdev_update` calls `thermal_cdev_set_cur_state` to set the new
      state
- `CONFIG_CPU_THERMAL`
  - cpufreq call `of_cpufreq_cooling_register` to register itself as a cooling
    device
  - when a thermal zone is heating up and trips, `cpufreq_set_cur_state` picks
    a lower freq
