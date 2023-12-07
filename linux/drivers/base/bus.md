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
