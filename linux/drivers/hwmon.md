Kernel hwmon
============

## Core

- `devm_hwmon_device_register_with_info` registers an `hwmon_chip_info`
  - other subsys can register an hwmon chip automatically as well
    - power supply core calls `power_supply_add_hwmon_sysfs`
    - thermal core calls `thermal_add_hwmon_sysfs`
    - nvme calls `nvme_hwmon_init`
    - ufs calls `ufs_hwmon_probe`
- `hwmon_chip_info` specifies ops and channel info
- userspace
  - lm-sensors, <https://github.com/lm-sensors/lm-sensors>
  - `libsensors` parses `/sys/class/hwmon`

## Channel Types and Attributes

- <https://docs.kernel.org/admin-guide/abi-testing-files.html#abi-file-testing-sysfs-class-hwmon>
- `hwmon_chip`
  - `hwmon_chip_update_interval` sample interval in ms
- `hwmon_temp`
  - `hwmon_temp_enable` enable/disable sensor
  - `hwmon_temp_input` temperature in mC
  - `hwmon_temp_type` sensor type (e.g., 1 for cpu)
  - `hwmon_temp_lcrit` low critical temp (too cold) in mC
  - `hwmon_temp_min` low temp (normal operaion) in mC
  - `hwmon_temp_max` high temp (normal operaion) in mC
  - `hwmon_temp_crit` high critical temp (too hot) in mC
  - `hwmon_temp_emergency` high emergency temp (way too hot) in mC
  - `hwmon_temp_*_alarm` low/high/critical/emergency temp detected
  - `hwmon_temp_label` suggested name/label for the sensor
  - `hwmon_temp_lowest` historical low temp
  - `hwmon_temp_highest` historical high temp
  - `hwmon_temp_reset_history` resets history
  - `hwmon_temp_rated_min` rated lowest temp
  - `hwmon_temp_rated_max` rated highest temp
  - `hwmon_temp_beep` beep
- `hwmon_in`
- `hwmon_curr`
- `hwmon_power`
- `hwmon_energy`
- `hwmon_energy64`
- `hwmon_humidity`
- `hwmon_fan`
- `hwmon_pwm`
- `hwmon_intrusion`
