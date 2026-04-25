# Kernel Platform Drivers

## `CONFIG_CHROME_PLATFORMS`

- these are chromebook drivers
- `CONFIG_CROS_EC` is the ec driver

## `CONFIG_X86_PLATFORM_DEVICES`

- there are mainly x86 laptop drivers

## `CONFIG_ARM64_PLATFORM_DEVICES`

- there are mainly snapdragon laptop drivers

## `CONFIG_ACPI_WMI`

- The ACPI WMI interface is a proprietary extension of the ACPI
  specification made by Microsoft to allow hardware vendors to embed WMI
  (Windows Management Instrumentation) objects inside their ACPI firmware.
  Typical functions implemented over ACPI WMI are hotkey events on modern
  notebooks and configuration of BIOS options.
