Kernel Device/Driver
====================

## `device_driver`

- ops
  - `probe`, called from `call_driver_probe`
    - `bus->probe` takes precedence
  - `sync_state`, called from `dev_sync_state`
    - `bus->sync_state` takes precedence
  - `remove`, called from `device_remove`
    - `bus->remove` takes precedence
  - `shutdown`, called from `device_shutdown`
    - `bus->shutdown` takes precedence
  - `suspend` and `resume` have no user?
    - they have been replaced by `pm`
  - `pm` is `dev_pm_ops`
  - `coredump`, called from `coredump_store`
    - this is triggered when `coredump` sysfs attr is written to
