Kernel ACPI Scan
================

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
    - `acpi_default_enumeration` calls `acpi_create_platform_device` to create
      a corresponding `platform_device`
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

## `/sys/devices/LNXSYSTM` on Dell Precision 5520

- these are ACPI devices and kernel drivers match for `modalias`
  - some drivers do `acpi_scan_add_handler` to create devices around `acpi_device`
  - some drivers drive the `acpi_dvice` directly
  - acpi core creates a `platform_device` for each `ACPI_TYPE_DEVICE` in
    `acpi_default_enumeration`
- LNXCPU:0[0-7], modalias LNXCPU, `CONFIG_ACPI_PROCESSOR`
- LNXPWRBN:00, modalias LNXPWRBN, `CONFIG_ACPI_BUTTON`
- LNXSYBUS:00, modalias LNXSYBUS
  - ACPI0003:00, modalias ACPI0003, `CONFIG_ACPI_AC`
  - ACPI000C:00, modalias ACPI000C, `CONFIG_ACPI_PROCESSOR_AGGREGATOR`
  - INT33A1:00, modalias INT33A1:PNP0D80, `CONFIG_INTEL_PMC_CORE`:`CONFIG_ACPI_SYSTEM_POWER_STATES_SUPPORT`
  - INT33D5:00, modalias INT33D5, `CONFIG_INTEL_HID_EVENT`
  - INT3400:00, modalias INT3400, `CONFIG_ACPI` and `CONFIG_INT340X_THERMAL`
  - INT340E:00, modalias INT340E:PNP0C02, `CONFIG_PCI_MMCONFIG`
  - MSFT0101:00, modalias MSFT0101, `CONFIG_TCG_CRB` and `CONFIG_TCG_TIS`
  - PNP0A08:00, modalias PNP0A08:PNP0A03, `CONFIG_PCI`
    - INT3403:06, modalias INT3403, `CONFIG_ACPI` and `CONFIG_INT340X_THERMAL`
    - LNXPOWER:0[0-2], modalias LNXPOWER
    - LNXVIDEO:00, modalias LNXVIDEO, `CONFIG_ACPI_VIDEO`
    - PNP0C02:0[2-5], modalias PNP0C02, `CONFIG_PCI_MMCONFIG`
    - PNP0C14:0[0-1], modalias PNP0C14, `CONFIG_ACPI_WMI`
    - device:0a/LNXVIDEO:01, modalias LNXVIDEO, `CONFIG_ACPI_VIDEO`
    - device:0f/DLL07BF:00, modalias DLL07BF:PNP0F13
    - device:0f/DLLK07BF:00, modalias DLLK07BF:PNP0303, `CONFIG_ACPI`
    - device:0f/INT0800:00, modalias INT0800
    - device:0f/INT3F0D:00, modalias INT3F0D:PNP0C02, `CONFIG_PCI_MMCONFIG`
    - device:0f/PNP0000:00, modalias PNP0000
    - device:0f/PNP0100:00, modalias PNP0100
    - device:0f/PNP0103:00, modalias PNP0103, `CONFIG_HPET`
    - device:0f/PNP0B00:00, modalias PNP0B00, `CONFIG_X86`
    - device:0f/PNP0C02:0[0-1], modalias PNP0C02, `CONFIG_PCI_MMCONFIG`
    - device:0f/PNP0C04:00, modalias PNP0C04
    - device:0f/PNP0C09:00, modalias PNP0C09, `CONFIG_ACPI`
      - INT3403:0[0-5], modalias INT3403, `CONFIG_ACPI` and `CONFIG_INT340X_THERMAL`
    - device:../device:../LNXPOWER:.. (total 20), modalias LNXPOWER
    - device:79/DLL07BF:01, modalias DLL07BF:PNP0C50, `CONFIG_I2C_HID`
  - PNP0C0A:00, modalias PNP0C0A, `CONFIG_ACPI_BATTERY`
  - PNP0C0C:00, modalias PNP0C0C, `CONFIG_ACPI_BUTTON`
  - PNP0C0D:00, modalias PNP0C0D, `CONFIG_ACPI_BUTTON`
  - PNP0C0E:00, modalias PNP0C0E, `CONFIG_ACPI_BUTTON`
  - PNP0C0F:0[0-7], modalias PNP0C0F, `CONFIG_PCI`
  - PNP0C14:0[2-3], modalias PNP0C14, `CONFIG_ACPI_WMI`
- LNXSYBUS:01, modalias LNXSYBUS
  - LNXTHERM:00, modalias LNXTHERM, `CONFIG_ACPI_THERMAL`

## Finding Drivers

- `find /sys/devices -name driver | xargs readlink -e | sort | uniq`
  - this lists all drivers that are bound to some devices
  - no unused driver
  - modules that discover and create the devices but are not themselves
    drivers are not included
- it is easy to find module names from here
  - some drivers does not have `module` links because they are built-in and
    they fail to set `device_driver::mod_name`
