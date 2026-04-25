# Kernel module

## Module

- device tables are extracted from .o by `scripts/mod/file2alias.c`.  They are
  parsed and aliases are added to the final `.ko`.

## configs and symbols

- `CONFIG_IKCONFIG`
  - embed the config in the kernel image/module
  - can be extracted using `./scripts/extract-ikconfig`
- `CONFIG_IKCONFIG_PROC`
  - make the embedded config accessible at `/proc/config.gz`
- `CONFIG_KALLSYMS`
  - embed symbols for pretty oops and backtraces
  - the same info is accessible at `/proc/kallsyms`

## `/sys/module`

- `param_sysfs_init`
  - `module_kset` is created with `kset_create_and_add`
  - `version_sysfs_builtin` adds built-in modules and their `version` for all
    built-in modules with `MODULE_VERSION`
  - `param_sysfs_builtin` adds built-in modules and their params for all
    built-in modules with `module_param_named`
    - `module_param_named` can specify to hide a param though
  - a built-in module without `MODULE_VERSION` or without visible params does
    not show up in `/sys/module`
- `init_module` syscall calls `load_module`
  - in `mod_sysfs_init` called from `mod_sysfs_setup`, the module kobj is
    added with `module_kset` as the kset
- when a module registers a driver, `module_add_driver` calls
  `module_create_drivers_dir` to add `/sys/module/<MODULE>/drivers`
  - unless it is a built-in module that does not set
    `device_driver::mod_name`, which is very common unfortunately
