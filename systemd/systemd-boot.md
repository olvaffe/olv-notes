systemd-boot
============

## Boot Protocol

- <https://uapi-group.org/specifications/specs/boot_loader_specification/>
  - `linux` specifies the kernel image
  - `initrd` specifies the initramfs
  - `options` specifies the cmdline
  - `devicetree` specifies the dtb
- `DEFINE_EFI_MAIN_FUNCTION(run, ...)` defines `run` as the entrypoint
- `config_load_all_entries` loads boot entries
  - `boot_entry_add_type1` parses an `<entry>.conf`
    - `linux` is parsed to `entry->type` and `entry->loader`
    - `initrd` is parsed to `entry->initrd`
    - `options` is parsed to `entry->options`
    - `devicetree` is parsed to `entry->devicetree`
- `image_start` boots the selected `BootEntry`
  - `initrd_prepare` loads the initramfs
  - `shim_load_image` loads the kernel
    - the kernel image is expected to be built with `CONFIG_EFI_STUB`
  - `devicetree_install` loads the dtb
  - `EFI_LOADED_IMAGE_PROTOCOL` is used to hand over the state to the kernel
