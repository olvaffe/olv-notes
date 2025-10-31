Kernel pstore
=============

## Overview

- there are backends and frontends
- backends are storage
  - `CONFIG_ACPI_APEI` uses acpi
  - `CONFIG_EFI_VARS_PSTORE` uses efivars
  - `CONFIG_PSTORE_RAM` uses reserved memory
    - `ramoops_probe` probes a `ramoops` platform device and calls
      `pstore_register`
    - the platform device is usually defined in dt
      - `ramoops_register_dummy` can add a platdev specified on cmdline
      - `CONFIG_CHROMEOS_PSTORE` can add a platdev specified in acpi
  - `CONFIG_PSTORE_BLK` uses reserved block device
- when `pstore_register` registers a backend, its flags decide which frontends
  are enabled
  - if `PSTORE_FLAGS_DMESG`, a kmsg dumper is registered to record last kmsg on panic
  - if `PSTORE_FLAGS_CONSOLE` and `CONFIG_PSTORE_CONSOLE`, a printk console is
    registered to record kmsg continuously
  - if `PSTORE_FLAGS_FTRACE` and `CONFIG_PSTORE_FTRACE`, a ftrace func is
    registered to record ftrace events
  - if `PSTORE_FLAGS_PMSG` and `CONFIG_PSTORE_PMSG`, `/dev/pmsg0` is created
    to record userspace message
