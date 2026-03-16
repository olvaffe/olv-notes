Kernel fwnode
=============

## Overview

- acpi fwnode
  - `acpi_bus_scan` builds a tree of `device`s under `/sys/devices/LNXSYSTM:00`
    - each `device` is an `acpi_device`
    - each `acpi_device` has a `fwnode_handle`
    - `acpi_init_device_object` calls `fwnode_init` with
      `acpi_device_fwnode_ops`
    - `acpi_init_device_object` also calls `acpi_init_properties` to scan
      `_DSD`
      - `acpi_nondev_subnode_extract` allocs a `acpi_data_node` which has a
      `fwnode_handle`
  - when `acpi_bus_scan` calls `acpi_bus_attach`,
    `acpi_create_platform_device` creates a platform device whose
    `pdev->dev.fwnode` points to the fwnode
  - when `pci_scan_single_device` scans a pci device, `pci_set_acpi_fwnode`
    calls `acpi_pci_find_companion` to find the companion acpi device
- dt fwnode
  - `of_fdt_unflatten_tree` builds a tree of `device_node` under
    `/sys/firmware/devicetree/base`
    - each `device_node` has a `fwnode_handle`
    - `of_node_init` calls `fwnode_init` with `of_fwnode_ops`
  - when `of_platform_default_populate` creates platform devices for device
    nodes, `of_device_alloc` calls `device_set_node` to set `dev->fwnode` and
    `dev->of_node`
- sw fwnode
  - when the driver expects fwnode, `device_add_software_node` can fake a
    fwnode
- `device_property_read_u32` calls `fwnode_property_read_int_array`
  - for acpi, `acpi_fwnode_property_read_int_array`
  - for dt, `of_fwnode_property_read_int_array`
  - for swnode, `software_node_read_int_array`
