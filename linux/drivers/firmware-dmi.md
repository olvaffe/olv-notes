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

## Initialization

- `dmi_setup` is called from x86 `setup_arch`
  - `dmi_scan_machine` finds and parses the smbios table in efi
    - `dmi_smbios3_present` and `dmi_decode` parse the table
    - `dmi_available` is set
  - `dmi_memdev_walk` parses `DMI_ENTRY_MEM_DEVICE`
- `dmi_init` creates `/sys/firmware/dmi`
