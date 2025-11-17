Linux MFD
=========

## Usage

- mfd driver calls `devm_mfd_add_devices` to add subdevs
  - driver typically defines a static const array of `mfd_cell`s
  - `mfd_add_device` calls `platform_device_add` to add a `mfd_cell` as a
    `platform_device`

## syscon

- syscon stands for system controller
  - it is a group of system registers that do not belong to any driver
  - there are usually several syscon devices in the DT
- `syscon_node_to_regmap` early returns unless the node has `syscon` in the
  compat string
- `of_syscon_register` registers a record for the node in `syscon_list`
  - `regmap_init_mmio` returns a `regmap`
- `syscon` platform driver
  - it matches the platform device that is named `syscon` exactly
  - it does not match OF node with `syscon` in the compat string
  - as such, it is rarely used
  - in fact, it has been removed since 6.14
