Kernel and Firmware
===================

## Initialization

- `firmware_init` creates `firmware_kobj`, `/sys/firmware`
  - `/sys/firmware/acpi` is from `acpi_init`
  - `/sys/firmware/dmi` is from `dmi_init`
    - it also creates `/sys/firmware/dmi/tables`
    - `dmi_sysfs_init` creates `/sys/firmware/dmi/entries`
  - `/sys/firmware/efi` is from `efisubsys_init`
  - `/sys/firmware/memmap` is from `firmware_memmap_init`

## `request_firmware`

- `_request_firmware_prepare` calls `firmware_request_builtin_buf` to search
  built-in firmwares first
- `fw_get_filesystem_firmware` tries to load from `/lib/firmware`
  - `kernel_read_file_from_path_initns` reads the entire file into a buffer
  - if compressed, the firmware is decompressed
- `firmware_fallback_platform`
  - this finds firmwares embedded in efi code
- `firmware_fallback_sysfs` is deprecated
  - e100 asks for `e100/d101m_ucode.bin`
  - udev receives an event and
    - `echo 1 > /sys/$DEVPATH/loading`
    . `cat $FIRMWARE > /sys/$DEVPATH/data`
    - `echo 0 > /sys/$DEVPATH/loading`

## `CONFIG_EXTRA_FIRMWARE`

- `CONFIG_EXTRA_FIRMWARE=foo.bin`
  - it generates `foo.bin.S` using `filechk_fwbin` macro
    - `incbin` is used to include `/lib/firmware/foo.bin`
    - a `struct builtin_fw` is emitted in `.builtin_fw` section
  - the ld script has `FW_LOADER_BUILT_IN_DATA` to collect all `struct
    builtin_fw` into an array denoted by `__start_builtin_fw` and
    `__end_builtin_fw`
- `firmware_request_builtin` goes through the `builtin_fw` array to find a
  match
  - the data is a direct pointer to the embedded firmware binary; no
    compression is allowed

## Firmware and KBuild

- It's easier to see an example, 9ac32e1bc0518b01b47dd34a733dce8634a38ed3
- `MODULE_FIRMWARE("firmware-name")` is added to the driver.
  - This is only informative.
- `fw-shipped-$(CONFIG_E100) += e100/d101m_ucode.bin` is added to
  `firmware/Makefile`.
- firmwares are put under `firmware/` in ihex or h16 forms
  - To convert ihex to bin, `objcopy -Iihex -Obinary`.
  - To convert ihex/h16 to fw, `ihex2fw`.
  - It is done automatically by the build system.
- If `CONFIG_FIRMWARE_IN_KERNEL` is defined,
  - firmware.bin is generated
  - firmware.gen.S is generated, which `.incbin` firmware.bin
  - It is compiled into firmware.gen.o.
  - All of firmware objects are compiled into the kernel.
    - They are in section `.builtin_fw` and are collected by
      `include/asm-generic/vmlinux.lds.h`
