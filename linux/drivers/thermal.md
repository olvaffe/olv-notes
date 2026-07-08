# Kernel Thermal

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

## OF Driver

- DT `thermal-zones`
  - `my-thermal`
    - `polling-delay` is the polling period in ms before tripping
      - it is 0 when tripping sends an interrupt and does not require polling
    - `polling-delay-passive` is the polling period in ms for passive cooling
      after tripping
      - it is never zero because the governor needs continuous samples to
        adjust throttling
    - `thermal-sensors` points to sensors
    - `trips`
      - `my-passive`
        - `temperature` is the trip temperature
        - `hysteresis` is the trip hysteresis
          - after tripping, it must drop to `temp minus hyst` to stop
        - `type` is `active`, `passive`, `hot`, `critical`
          - `THERMAL_TRIP_ACTIVE` triggers active cooling
          - `THERMAL_TRIP_PASSIVE` triggers passive cooling
          - `THERMAL_TRIP_HOT` triggers `tz->ops.hot`, which is typically NULL
          - `THERMAL_TRIP_CRITICAL` triggers `tz->ops.critical`, which defaults
            to `thermal_zone_device_critical` to shut down the system
    - `cooling-maps`
      - `map0`
        - `trip` points to the trip
        - `cooling-device` points to the cdevs
- when the sensor driver probes,
  - `devm_thermal_of_zone_register` registers a zone for each sensor channel
  - `thermal_zone_device_enable` enables a zone
  - `thermal_add_hwmon_sysfs` registers the zone as a hwmon dev
- `devm_thermal_of_zone_register`
  - `of_thermal_zone_find` returns the tz node
    - it walks through child nodes of `thermal-zones` and matches
      `thermal-sensors`
  - `thermal_of_trips_init` creates trips
    - `thermal_of_populate_trip` creates a `thermal_trip` for each child node
      of `trips`
      - `trip->temperature` is from `temperature`
      - `trip->hysteresis` is from `hysteresis`
      - `trip->type` is from `type`
  - `thermal_of_monitor_init` parses polling periods
  - `thermal_of_parameters_init` parses parameters
    - `tzp->no_hwmon` is true
    - `tzp->sustainable_power` defaults to false
    - `tzp->slope` defaults to 1
    - `tzp->offset` defaults to 0
  - `of_ops.should_bind` is set to `thermal_of_should_bind`
    - it returns true when the cdev is in `cooling-maps`
  - `thermal_zone_device_register_with_trips` registers the zone
    - it allocates a `thermal_zone_device` and inits to what we have parsed
      from dt
    - `thermal_zone_device_init` further inits tz
      - `tz->temperature` is `THERMAL_TEMP_INIT`
      - all trips are moved from `tz->trips_invalid` to `tz->trips_high`,
        because they are higher than `THERMAL_TEMP_INIT`
    - `thermal_zone_init_governor`
      - `tz->governor` is `def_governor`, which is often `thermal_gov_step_wise`
    - `thermal_zone_create_device_groups` inits `tz->device.groups` for sysfs
    - `thermal_zone_init_complete`
      - it adds the new tz to global `thermal_tz_list`
      - `__thermal_zone_cdev_bind` binds cdevs that are on `thermal_cdev_list`
        and are in `cooling-maps`
        - it creates and adds a `thermal_instance` to `td->thermal_instance`
          and `cdev->thermal_instances`
      - `tz->state` becomes `TZ_STATE_READY`
      - `__thermal_zone_device_update` updates
    - `thermal_notify_tz_create`
    - `thermal_debug_tz_add` inits `tz->debugfs` for debugfs
  - `thermal_zone_device_enable` enables the zone
    - `tz->mode` is set to `THERMAL_DEVICE_ENABLED`
    - `__thermal_zone_device_update` updates
    - `thermal_notify_tz_enable` sends `THERMAL_GENL_EVENT_TZ_ENABLE`
- `__thermal_zone_device_update`
  - it is called on any major change
    - `enum thermal_notify_event` lists the reasons
  - `__thermal_zone_get_temp` calls `tz->ops.get_temp` to get cur temp
  - `tz->last_temperature` and `tz->temperature` are updated
  - `trace_thermal_temperature` sends `thermal/thermal_temperature`
  - `thermal_genl_sampling_temp` sends `THERMAL_GENL_SAMPLING_TEMP`
  - `tz->notify_event` is set
  - `thermal_zone_handle_trips`
    - it loops through `tz->trips_reached` and calls `thermal_trip_crossed` if
      `tz->temperature` is below `td->threshold`
    - it loops through `tz->trips_high` and calls `thermal_trip_crossed` if
      `tz->temperature` is above `td->threshold`
    - `low` is set to temp of last trip of `tz->trips_reached`
    - `high` is set to temp of first trip of `tz->trips_high`
  - `thermal_thresholds_handle`
    - `thermal_threshold_find_boundaries` updates `low` and `hight` based on
      user thresholds
    - if user thresholds crossed, it sends either
      - `THERMAL_GENL_EVENT_THRESHOLD_UP` or
      - `THERMAL_GENL_EVENT_THRESHOLD_DOWN`
  - `thermal_zone_set_trips`
    - `tz->prev_low_trip` is set to `low`
    - `tz->prev_high_trip` is set to `high`
    - `tz->ops.set_trips` programs the hw for the new trip points
      - that is, when to send hw interrupts
  - `governor->manage` is typically `step_wise_manage`
  - `thermal_debug_update_trip_stats` updates stats for debugfs
  - `monitor_thermal_zone` sets up polling if necessary
- `step_wise_manage`
  - `thermal_zone_trip_update`
    - `get_tz_trend` returns the trend
      - `THERMAL_TREND_STABLE`
      - `THERMAL_TREND_RAISING`
      - `THERMAL_TREND_DROPPING`
    - `get_target_state` returns the new cdev target state
  - `thermal_cdev_update` calls `thermal_cdev_set_cur_state`
    - `cdev->ops->set_cur_state` applies the new target state
    - `thermal_notify_cdev_state_update` sends
      `THERMAL_GENL_EVENT_CDEV_STATE_UPDATE`
    - `thermal_cooling_device_stats_update` updates cdev sysfs stats
    - `thermal_debug_cdev_state_update` updates cdev debugfs stats
