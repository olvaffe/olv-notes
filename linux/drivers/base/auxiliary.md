Kernel and auxiliary
====================

## Overview

- when a device consists of logically different ip blocks, it can create
  multiple auxiliary devices to be driven by multiple auxiliary drivers
  - the drivers can belong to the same or different subsystems
- during init, `driver_init` calls `auxiliary_bus_init` to register the bus
- `auxiliary_device_init` inits a `auxiliary_device` struct
  - both `dev->parent` and `auxdev->name` must be filled in
- `auxiliary_device_add` adds the `auxiliary_device` to the bus
  - `dev_set_name` sets the name to `KBUILD_MODNAME.auxdev->name.auxdev->id`
- `module_auxiliary_driver` or `auxiliary_driver_register` registers a driver
  - driver name is set to `KBUILD_MODNAME.auxdrv->name`
- `auxiliary_match` matches dev/drv
  - it matches `dev_name(dev)` against the id table, with `auxdev->id` omitted
- `auxiliary_probe` probes dev/drv
