Kernel module
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

## configs and symbols

- `CONFIG_IKCONFIG`
  - embed the config in the kernel image/module
  - can be extracted using `./scripts/extract-ikconfig`
- `CONFIG_IKCONFIG_PROC`
  - make the embedded config accessible at `/proc/config.gz`
- `CONFIG_KALLSYMS`
  - embed symbols for pretty oops and backtraces
  - the same info is accessible at `/proc/kallsyms`
