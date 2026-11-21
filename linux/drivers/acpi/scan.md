# Kernel ACPI Scan

## Bus Scan

- `acpi_init` calls `acpi_scan_init`
  - it adds various handlers for scanned acpi devices
    - `acpi_pci_root_init` adds `pci_root_handler`
      - it matches PCI host controllers.  `acpi_pci_root_add` calls
        `pci_acpi_scan_root` to createa a `pci_bus`.
    - `acpi_pci_link_init` adds `pci_link_handler`
    - `acpi_processor_init` adds `processor_handler` and `processor_container_handler`
    - `acpi_lpss_init` adds `lpss_handler`
      - it matches LPSS devices.  `acpi_lpss_create_device` calls
        `acpi_create_platform_device` to create a `platform_device` for the
        `acpi_device`
    - more
  - `acpi_bus_scan(ACPI_ROOT_OBJECT)` performs the scan
- `acpi_bus_scan` adds ACPI device node objects under a given handle
  - pass 1 calls `acpi_bus_check_add_1` on each acpi handle to create acpi dev
    - in this pass, if `acpi_scan_check_dep` returns true, it is skipped
    - `acpi_add_single_object` adds an `acpi_device`
      - `acpi_init_device_object` initializes the `acpi_device`
        - `dev->parent` is set to `acpi_bus_get_parent`, which returns NULL for
          `ACPI_ROOT_OBJECT`
        - `dev->pnp.ids` is set up by `acpi_set_pnp_ids`
      - `acpi_device_add` adds the `acpi_device`
        - device name is set to `acpi_device_hid` which returns the first name
          of `device->pnp`
  - pass 1 calls `acpi_bus_attach` to handle acpi devices
    - if matched by handler, `acpi_scan_attach_handler` attaches the handler
      to the adev
    - for most acpi devices, `acpi_default_enumeration` calls
      `acpi_create_platform_device` to create platdevs
  - pass 2 calls `acpi_scan_postponed`
    - it repeats pass 1 for devices with deps
    - these are devices on ec or pci, which are postponed until
      `acpi_dev_clear_dependencies` clears their deps

## `acpi_device`

- `acpi_add_single_object` creates an `acpi_device` for an `acpi_handle`
  - `acpi_init_device_object` initializes the struct
    - `device->dev.bus` is set to `acpi_bus_type`
    - `fwnode_init` initializes `device->fwnode` with
      `acpi_device_fwnode_ops`
    - `acpi_set_pnp_ids` adds various ids to `device->pnp`
      - if it is `ACPI_ROOT_OBJECT`, the hid is `ACPI_SYSTEM_HID` (`LNXSYSTM`)
      - if it is a device, it invokes `acpi_get_object_info` and adds `_HID`,
        `_CID`, etc.
    - `acpi_init_properties`
  - if the type is a device or a processor, `acpi_scan_init_status`
    initializes the status
    - `acpi_bus_get_status_handle` evalutes `_STA` of the `acpi_handle`
    - `acpi_set_device_status` saves the status in `acpi_device`
  - `acpi_bus_get_power_flags` initializes the power flags
    - it checks/evaluates `_PS0`, etc.
  - `acpi_bus_get_wakeup_device_flags` initializes the wakeup flags
    - it checks/evaluates `_PRW`, etc.
  - `acpi_tie_acpi_dev` calls `acpi_attach_data` to associate the
    `acpi_device` with the `acpi_handle`
  - `acpi_device_add` adds the device to the acpi bus
