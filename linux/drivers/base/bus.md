Kernel Driver Bus
=================

## Initialization

- `buses_init` is called from `driver_init` during driver subsystem init
- `bus_kset` is created with `kset_create_and_add` with no parent kobject
  - it appears in `/sys/bus`
- `system_kset` is created with `kset_create_and_add` under `devices_kset`
  - it appears in `/sys/devices/system`

## Bus Registration

- `bus_register` registers a new `bus_type` under `/sys/bus`
  - a `subsys_private` is allocted to represent the bus
    - `priv->bus` points to the specified `bus_type`
    - `priv->subsys` is the kset of the bus (`/sys/bus/<foo>`)
    - `priv->devices_kset` is the kset of devices (`/sys/bus/<foo>/devices`)
    - `priv->drivers_kset` is the kset of drivers (`/sys/bus/<foo>/drivers`)
    - `priv->drivers_autoprobe` is true by default
  - these attributes are added to the bus using `bus_create_file`
    - `bus_attr_uevent` for `uevent`
    - `bus_attr_drivers_probe` for `drivers_probe`
    - `bus_attr_drivers_autoprobe` for `drivers_autoprobe`
- `subsys_system_register` registers a new `bus_type`
  - it calls `bus_register` to register a bus (`/sys/bus/<foo>`)
  - a `struct device` is allocated and is registered as well
    (`/sys/devices/system/<foo>`)
- `subsys_virtual_register` register a new `bus_type`
  - it is similar to `subsys_system_register` except the device is under
    `/sys/device/virtual/<foo>`
- `subsys_interface_register` registers a new `subsys_interface` to a bus
  - it is added to `sp->interfaces`
  - `subsys_interface` has `add_dev` and `remove_dev` callbacks that are
    called when devices are added to and removed from the bus

## Bus Manipulation

- `bus_add_device` adds a `device` to its `dev->bus` (if any)
  - this adds `bus->dev_groups` to the device
  - this creates a symlink under `/sys/bus/<foo>/devices` to the device
  - this also creates a symlink, `subsystem`, pointing back to the bus
- `bus_add_driver` adds a `device_driver` to its `drv->bus`
  - this allocates a `driver_private` to represent the driver
    - it is to provide `/sys/bus/<foo>/<drv>`
  - `driver_attach` tries to match the driver to a device on the bus
  - `module_add_driver` creates files to associate a driver and a module (if
    any)
    - `/sys/bus/<foo>/drivers/<drv>/module` points to the module
    - `/sys/modules/<mod>/drivers/<drv>` points to the driver
  - these attributes are added to the driver
    - `driver_attr_uevent`
    - `bus->drv_groups`
    - `driver_attr_bind`
    - `driver_attr_unbind`

## `bus_type` ops

- `match`, called from `driver_match_device`
  - this is used to check if a dev/drv pair matches during probe
- `uevent`, called from `dev_uevent`
  - this adds bus-specific uevents
- `probe`, called from `call_driver_probe`
  - bus probe takes precedence over `drv->probe` during probe
    - the idea it to perform bus-specific setup before calling `drv->probe`
- `sync_state`, called from `dev_sync_state`
  - this seems to be used by device links
- `remove`, called from `device_remove`
  - this is called when a device is removed
- `shutdown`, called from `device_shutdown`
  - this is called when the system is about to reboot
- `online`, called from `device_online`
  - this brings an offline device back online
- `offline`, called from `device_offline`
  - this is called in preparation for hot removal
- `suspend`, called from `device_suspend`
  - this is called when the system is about to suspend
  - this is legacy and `pm` takes precedence
- `resume`, called from `device_resume`
  - this is called when the system is just resumed
  - this is legacy and `pm` takes precedence
- `num_vf`, called from `dev_num_vf`
  - this is only used by rtnetlink
- `dma_configure`, called from `really_probe`
  - this is called during probe
- `dma_cleanup`, called from `__device_release_driver`
  - this is called during unbind
- `pm` is bus-specific `dev_pm_ops`
  - it takes precedence over driver's

## Autoprobe

- because `priv->drivers_autoprobe` is true by default,
  - when `device_add` adds a device, it calls `bus_probe_device` which calls
    `device_initial_probe`
  - when `driver_register` registers a driver, it calls `bus_add_driver` which
    calls `driver_attach`
  - in both cases, `driver_probe_device` is called on each dev/drv pair
