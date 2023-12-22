Kernel ACPI
===========

## Overview

- <https://www.kernel.org/doc/html/latest/firmware-guide/acpi/index.html>
- `osl.c` implements `acpiosxf.h`, an abstraction interface acpica uses to
  communicate with the os
- `osi.c` manipulates the `_OSI` table
  - bios has a built-in `_OSI` table for the supported operating systems
  - `acpi_osi_setup_late` calls `acpi_install_interface` and
    `acpi_remove_interface` to update the table
    - e.g., removes `_OSI(Windows 2012)` to disable some paths in acpi

## ACPICA

- the repo is at <https://github.com/acpica/acpica>
  - `generate/linux/linuxize.sh` copies a subset of acpica files to the kernel
    and applies a coding style transformation
  - looking at `linux_dirs` in `generate/linux/libacpica.sh`, they are copied
    to these directories
    - `drivers/acpi/acpica`
    - `include/acpi`
    - `include/acpi/platform`
    - `tools/power/acpi/common`
    - `tools/power/acpi/os_specific/service_layers`
    - `tools/power/acpi/tools/acpidump`
- `include/acpi/acpi.h` is the master header file for acpica
  - other than types and macros, these sub-headers also declare functions
  - `platform/aclinuxex.h` declares `acpi_os_*`
    - they appear to be an os abstraction layer for acpica
    - acpica calls into kernel using this layer
  - `acpiosxf.h` declares `acpi_os_*`
    - they appear to be an os abstraction layer for userspace tools
    - acpica userspace tools calls into this layer, wher
      `tools/power/acpi/os_specific/service_layers` provides an impl
  - `acpixf.h` declares `acpi_*`
    - they are the interface defined by acpica
    - kernel calls into acpica using this interface
    - acpica uses this interface internally as well
- `acpixf.h` interface
  - initialization, such as
    - `acpi_initialize_tables`
    - `acpi_initialize_subsystem`
    - `acpi_enable_subsystem`
    - `acpi_initialize_objects`
  - misc global, such as
    - `acpi_enable`
    - `acpi_disable`
    - `acpi_install_interface`
  - table load/unload, such as
    - `acpi_install_table`
    - `acpi_load_table`
    - `acpi_load_tables`
  - table manipulation, such as
    - `acpi_reallocate_root_table`
    - `acpi_get_table_header`
    - `acpi_get_table`
  - namespace and name, such as
    - `acpi_walk_namespace`
    - `acpi_get_devices`
    - `acpi_get_name`
    - `acpi_get_handle`
    - `acpi_attach_data`
  - object manipulation and enumeration, such as
    - `acpi_evaluate_object`
    - `acpi_get_object_info`
    - `acpi_get_next_object`
    - `acpi_get_type`
    - `acpi_get_parent`
  - handler, such as
    - `acpi_install_global_event_handler`
    - `acpi_install_fixed_event_handler`
    - `acpi_install_gpe_handler`
    - `acpi_install_notify_handler`
    - `acpi_install_address_space_handler`
    - `acpi_install_interface_handler`
  - lock, such as
    - `acpi_acquire_global_lock`
    - `acpi_acquire_mutex`
  - fixed event, such as
    - `acpi_enable_event`
    - `acpi_disable_event`
    - `acpi_clear_event`
    - `acpi_get_event_status`
  - general purpose event (GPE), such as
    - `acpi_enable_gpe`
    - `acpi_disable_gpe`
    - `acpi_get_gpe_status`
  - resource, such as
    - `acpi_walk_resources`
  - hw (acpi device)
    - `acpi_read`
    - `acpi_write`
  - sleep/wake
    - `acpi_enter_sleep_state`
    - `acpi_leave_sleep_state`
  - logging
    - `acpi_error` and `ACPI_ERROR`
    - `acpi_warning` and `ACPI_WARNING`
    - `acpi_info` and `ACPI_INFO`
    - `acpi_debug_print` and `ACPI_DEBUG_PRINT`
    - these are mainly called from acpica into the kernel
