ACPI
====

## Bus Scan

- `acpi_init` calls `acpi_scan_init` which calls
  `acpi_bus_scan(ACPI_ROOT_OBJECT)`
- `acpi_bus_scan` adds ACPI device node objects under a given handle
  - `acpi_bus_type_and_status` returns the type and status of the given handle
  - `acpi_add_single_object` creates and adds an `acpi_device` for the handle
    - `acpi_init_device_object` initializes the `acpi_device`
      - `dev->parent` is set to `acpi_bus_get_parent`, which returns NULL for
      	`ACPI_ROOT_OBJECT`
      - `dev->pnp` is set up by `acpi_set_pnp_ids`, which has
      	`ACPI_SYSTEM_HID` (LNXSYSTM) for `ACPI_ROOT_OBJECT`
    - `acpi_device_add` adds the `acpi_device`
      - device name is set to `acpi_device_hid` which returns the first name
      	of `device->pnp`
  - `acpi_walk_namespace` recursively walks the namspace and adds all
    `acpi_device` under `ACPI_ROOT_OBJECT`
  - `acpi_bus_attach` recursively calls `acpi_scan_attach_handler` on each
    `acpi_device`
    - this calls the attach callback of the matching `acpi_scan_handler`s
- before scanning the root object, scan handlers are registered with
  `acpi_scan_add_handler`
  - `lpss_handler` matches LPSS devices.  `acpi_lpss_create_device` calls
    `acpi_create_platform_device` to create a `platform_device` for the
    `acpi_device`
  - `pci_root_handler` matches PCI host controllers.  `acpi_pci_root_add`
    calls `pci_acpi_scan_root` to createa a `pci_bus`.
- there are `acpi_driver`s that are registered to acpi bus to drive
  `acpi_device` directly using `acpi_bus_register_driver`
- but more commonly, `acpi_scan_handler` is used to set up devices from
  `acpi_device`s
  - e.g., a `platform_device` can be set up from an `acpi_device`, and is
    driven driven by a `platform_driver`
