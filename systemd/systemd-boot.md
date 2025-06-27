systemd-boot
============

## The Boot Loader Specification

- The Partitions
  - prefer ESP that is at least 1GB, mounted to `/boot`
  - if not possible, create XBOOTLDR that is at least 1GB
    - XBOOTLDR is mounted to `/boot`
    - ESP is mounted to `/efi` in this case
  - this spec only defines semantics for
    - `$BOOT/EFI/Linux/`
    - `$BOOT/loader/entries/`
    - `$BOOT/loader/entries.srel`
- Boot Loader Entries
  - type #1: `$BOOT/loader/entries/<entry>.conf`
    - `title` is human-readable title
    - `version` is human-readable version
    - `machine-id` is machine id
    - `sort-key` is string used for sorting
    - `linux` is kernel path
    - `initrd` is zero or more initrd paths
    - `efi` is efi app path (usually uki)
    - `options` is zero or more cmdlines
    - `devicetree` is dtb path
    - `devicetree-overlay` is separate-separated zero or more dtbo paths
    - `architecture` is EFI arch (`x64`, `aa64`, etc.)
  - `$BOOT/loader/entries.srel` should be `type1`
  - type #2: `$BOOT/EFI/Linux/<uki>.efi`
    - it is self-contained
- Locating Boot Entries
  - boot loader populates its boot menu with
    - type #1 entries
    - type #2 entries
    - others
- Boot counting
  - a boot entry named `<entry>+N-M.conf` or `<uki>+N-M.efi` is boot counted
  - boot loader decrements N and increments M on each boot attempt
    - N is tries left
    - M is tries attempted
  - if a boot attempt is considered successful, os drops the counts from the
    file name
- Sorting
  - sort by `sort-key`, `machine-id`, `version`, and then file names
    - `title` should not be used

## Internals

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
