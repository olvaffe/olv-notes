Kernel DMI
==========

## SMBIOS

- DMI, Desktop Management Interface, exposes system data including SMBIOS
  - SMBIOS was originally known as DMIBIOS
- 1.0, by Phoenix
- 2.0, 1996, by multiple bios vendors
- 2.3, 1998
- 2.3.1, 1999, by DMTF
- 3.0.0, 2015
- 3.4.0, 2020
- 3.5.0, 2021
- 3.6.0, 2022
- 3.7.0, 2023
- table entries
  - BIOS Information: vendor, version, date, etc.
  - System Information: manufacturer, product name, serial, sku, etc.
  - Base Board Information: manufacturer, product name, asset tag, etc.
  - Chassis Information: manufacturer, type, serial, asset tag, etc.
  - Processor Information: socket, cpu family/model/stepping, etc.
  - Cache Information: l1 size, l2 size, l3 size, etc.
  - Port Connector Information: dp, usb, audio jack, etc.
  - System Slot Information: pcie
  - On Board Device Information: igpu
  - OEM Strings: oem info
  - Memory Device: mem speed, etc.
  - Portable Battery: manufacturer, capacity, voltage, etc.

## Initialization

- `dmi_setup` is called from x86 `setup_arch`
  - `dmi_scan_machine` finds and parses the smbios table in efi
    - `dmi_smbios3_present` and `dmi_decode` parse the table
    - `dmi_available` is set
  - `dmi_memdev_walk` parses `DMI_ENTRY_MEM_DEVICE`
- `dmi_init` creates `/sys/firmware/dmi`
