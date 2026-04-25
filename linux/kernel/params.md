# Kernel params

## Parameters

- `init.h`
  - `__setup_param` / `__setup` / `early_param`
  - for really core code
  - ignored if built as module
  - they are collected to `__setup_start` and `__setup_end` array
  - `do_early_param` or `obsolete_checksetup` parses them as unknown options
- `moduleparam.h`
  - `module_param` for simple params
    - `module_param_cb` to handle complex params
  - `<level>_param_cb` are evaluated before initcall `<level>`
    - e.g., `usbcore.quirks` is a `device_param_cb`
  - `core_param` is similar to `__setup`
    - built-in only
    - for transitioning from `__setup`
    - unlike `module_param`, the param is not prefixed by
      `MODULE_PARAM_PREFIX`
  - they are collected to `__start___param` and `__stop___param` array
  - `parse_args` parses them
- `/sys/module/*/parameters` for most parameters
  - `module_param` can control whether a param is visible in sysfs or not
- interesting parameters
  - `initcall_debug=0`
  - `init=/sbin/init`
  - `rdinit=/init`
  - `debug` sets console log level to 10
  - `quiet` sets console log level to `CONFIG_CONSOLE_LOGLEVEL_QUIET` (4)

## `module_param`

- e.g., `module_param_named(debug, __drm_debug, ulong, 0600);`
- `param_check_ulong` generates a unused function to check that `__drm_debug`
  is indeed `unsigned long`
- `module_param_cb` defines a `kernel_param`
  - `name` points to static string `drm.debug`
  - `mod` is `THIS_MODULE`
  - `ops` is `param_ops_ulong`
    - this is defined by `STANDARD_PARAM_DEF`
  - `perm` is 0600
  - `level` is -1
    - it is parsed at the same time as `__setup`, not delayed until before the
      corresponding initcall
  - `flags` is 0
  - `arg` is `&__drm_debug`
- when `parse_args` calls `parse_one` to parse `drm.debug=0xf`,
  - it calls `param_set_ulong` defined by `STANDARD_PARAM_DEF`
  - `kstrtoul` parses `0xf` str to `__drm_debug`
- if drm module is built-in, `param_sysfs_builtin` creates
  `/sys/module/drm/parameters/debug`
  - `lookup_or_create_module_kobject` creates `/sys/module/drm`
  - `add_sysfs_param` re-creates `/sys/module/drm/parameters` to add `debug`
