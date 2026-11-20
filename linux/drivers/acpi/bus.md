# Kernel ACPI Bus

## Initialization

- x86 `start_kernel -> setup_arch`
  - `efi_init -> efi_config_init -> efi_config_parse_tables` parses efi tables
  - `acpi_boot_table_init -> acpi_locate_initial_tables -> acpi_initialize_tables`
    - `acpi_os_get_root_pointer` returns root pointer from efi
    - `acpi_tb_parse_root_table` parses root table
  - `early_acpi_boot_init` parses `BOOT` and `MADT`
  - `acpi_boot_init` parses `BOOT`, `FADT`, `MADT`, `HPET`, `SPCR`, etc.
- arm `start_kernel -> setup_arch`
  - `efi_init -> efi_config_init -> efi_config_parse_tables` parses efi tables
  - `acpi_boot_table_init`
    - `acpi_table_init -> acpi_locate_initial_tables ->
      acpi_locate_initial_tables -> acpi_initialize_tables` is similar to x86
    - `acpi_fadt_sanity_check` parses `FADT`
    - `acpi_parse_spcr` parses `SPCR`
- `start_kernel`
  - `acpi_early_init` initializes acpica
    - it calls `acpi_reallocate_root_table` and `acpi_initialize_subsystem`
      defined by acpica
  - `acpi_subsystem_init`
- `subsys_initcall(acpi_init)` inits acpi subsys
  - `acpi_kobj` is created at `/sys/firmware/acpi`
  - `init_prmt` parses `PRMT` (platform runtime mechanism table)
    - this appears to provide additional uefi runtime services that are
      available to the kernel and the acpi
    - when acpi invokes such a service, I guess it traps to
      `acpi_platformrt_space_handler` and let `efi_call_acpi_prm_handler`
      handle the invocation
  - `acpi_bus_init` initializes the kernel acpi bus
  - `acpi_scan_init` initiates the acpi scan
  - many more to parse various acpi tables
- `acpi_bus_init` initializes acpi bus
  - `acpi_os_initialize1` starts `kacpid` wq and calls `acpi_osi_init` to
    update `_OSI` table
  - it calls these functions defined by acpica
    - `acpi_load_tables`
    - `acpi_enable_subsystem`
    - `acpi_initialize_objects`
    - `acpi_install_table_handler`
  - `acpi_sysfs_init` populates `/sys/firmware/acpi`
    - `acpi_tables_sysfs_init` populates `tables`
  - `acpi_sleep_init` initializes acpi-based sleep
  - `acpi_bus_init_irq` invokes `_PIC`
  - `acpi_root_dir` represents `/proc/acpi`
  - `companion_bus_register` registers `acpi_bus_type`
    - there is no direct acpi driver

## `acpi_bus_type`

- there are a lot of `acpi_device`
  - acpi bus scan calls `acpi_device_add` to add them
  - `ACPI_ROOT_OBJECT` is at `/sys/devices/LNXSYSTM:00`
  - each `acpi_device` also embeds a `fwnode_handle` with
    `acpi_device_fwnode_ops`
- there is no `acpi_driver` in modern kernel
- other bus drivers are used to drive these devices
  - e.g., acpi bus scan calls `acpi_create_platform_device` to create platform
    devices for some of the acpi devices
  - `platform_match`, the match function of the platform bus, calls
    `acpi_driver_match_device` to match with `acpi_match_table`
  - when a platform driver probes a platform device, it can use
    `device_get_match_data` to get the associated data
    - the platform device's `fwnode` points to the one embedded in the acpi device
    - `device_get_match_data` calls `acpi_fwnode_device_get_match_data`

## OF Match

- `acpi_bus_match` calls `__acpi_match_device` to match driver/device
- if the device has the special ID `ACPI_DT_NAMESPACE_HID` (`PRP0001`),
  `acpi_of_match_device` matches using the device tree compatible string
  instead
