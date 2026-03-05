Kernel Device/Driver
====================

## Manual Bind/Unbind

- manual bind/unbind
  - `bind_store`
    - `bus_find_device_by_name` finds the dev
    - `driver_match_device` checks against the id table, etc.
    - `device_driver_attach` attaches the drv to the dev
  - `unbind_store`
    - `bus_find_device_by_name` finds the dev
    - `device_driver_detach` detaches the drv to the dev
- `device_driver_attach` calls `__driver_probe_device`
  - `pm_runtime_get_sync` resumes all suppliers and the parent
  - `pm_runtime_barrier`
  - `really_probe`
    - `device_links_check_suppliers` checks that all suppliers have drivers
    - `pinctrl_bind_pins` configs pinctrl, if any
    - `dev->bus->dma_configure` configs iommu, etc., if any
    - `driver_sysfs_add` adds symlinks between dev and drv
    - `dev->pm_domain->activate` configs pmdomain, if any
    - `call_driver_probe` calls `drv->probe`
    - `device_add_groups` adds sysfs attr groups
    - `pinctrl_init_done` configs pinctrl, if any
    - `dev->pm_domain->sync` configs pmdomain, if any
    - `driver_bound` advertises the dev-drv bound to consumers, userspace,
      etc.
  - `pm_request_idle`
  - `pm_runtime_put` suspends all suppliers and the parent
- `device_driver_detach` calls `__device_release_driver`
  - `pm_runtime_get_sync` resumes the dev
  - `device_links_unbind_consumers` unbinds consumer drivers
  - `driver_sysfs_remove` undoes `driver_sysfs_add`
  - `pm_runtime_put_sync` suspends the dev
  - `device_remove` calls `drv->remove`
  - ` dev->bus->dma_cleanup` undoes `dev->bus->dma_configure`
  - `device_unbind_cleanup`
  - `device_links_driver_cleanup` advertises the dev-drv unbound to producers,
    etc.

## Auto Bind/Unbind

- when a driver is added, `driver_attach` matches and probes all devices on a
  bus
  - `driver_match_device` checks against the id table, etc.
  - `driver_probe_device` probes the device, with `-EPROBE_DEFER` fallback
  - if async allowed, `__driver_attach_async_helper` calls
    `driver_probe_device` async instead
- when a driver is removed (e.g., module unload), `driver_detach` detaches the
  driver from all devices
- when a device is added, `device_initial_probe` finds the matching driver to
  probe the device
- when a device is removed, `device_release_driver` detaches the driver from
  the device
- `device_attach` is rarely used and is similar to `device_initial_probe`
  - `drivers_probe_store` or `device_reprobe` triggers the call

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
