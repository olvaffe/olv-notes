Kernel Debug
============

## Config

- enable `CONFIG_DYNAMIC_DEBUG` for all `pr_debug` messages
  - dynamically configurable
  - `dyndbg="+p"` for cmdline
  - `/sys/kernel/debug/dynamic_debug/control`
- statically enabling `pr_debug` messages
  - `CONFIG_DEBUG_DRIVER` for driver core
  - `CONFIG_DEBUG_KOBJECT` for kobject
  - `CONFIG_PCI_DEBUG` for PCI
  - `CONFIG_DEBUG_GPIO` for GPIO
  - `CONFIG_I2C_DEBUG_{CORE,ALGO,BUS}` for I2C
  - `CONFIG_HWMON_DEBUG_CHIP` for hardware monitor
  - `CONFIG_DEBUG_PINCTRL` for pin control
  - `CONFIG_POWER_SUPPLY_DEBUG` for power supply
  - `CONFIG_REGULATOR_DEBUG` for regulator
  - `CONFIG_RTC_DEBUG` for RTC
  - `CONFIG_SPI_DEBUG` for SPI

## Parameters

- `init.h`
  - `__setup_param` / `__setup` / `early_param`
  - for really core code
  - ignored if built as module
- `moduleparam.h`
  - `module_param` for simple params
    - `module_param_cb` to handle complex params
  - `<level>_param_cb` are evaluated before initcall `<level>`
    - e.g., `usbcore.quirks` is a `device_param_cb`
  - `core_param` is similar to `__setup`
    - built-in only
    - for transitioning from `__setup`
- `/sys/module/*/parameters` for most parameters
  - `module_param` can control whether a param is visible in sysfs or not
- interesting parameters
  - `initcall_debug=0`
  - `init=/sbin/init`
  - `rdinit=/init`
  - `debug` sets console log level to 10
  - `quiet` sets console log level to `CONFIG_CONSOLE_LOGLEVEL_QUIET` (4)

## netconsole

- specify `netconsole=@192.168.0.2/,@192.168.0.1/` in cmdline
- host does `nc -u -l 6666`
