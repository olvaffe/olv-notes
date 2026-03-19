Kernel params
=============

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
