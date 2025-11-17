Kernel Platform Drivers
=======================

## Platform support for Chrome hardware

- these are chromebook drivers
- `CONFIG_CROS_EC` is the ec driver

## X86 Platform Specific Device Drivers

- there are mainly x86 laptop drivers
- `CONFIG_ACPI_WMI`
  - The ACPI WMI interface is a proprietary extension of the ACPI
    specification made by Microsoft to allow hardware vendors to embed WMI
    (Windows Management Instrumentation) objects inside their ACPI firmware.
    Typical functions implemented over ACPI WMI are hotkey events on modern
    notebooks and configuration of BIOS options.

## ARM64 Platform-Specific Device Drivers

- there are mainly snapdragon laptop drivers
