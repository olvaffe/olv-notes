Chrome OS AP Firmware
=====================

## AP firmware (BIOS)

- depthcharge lives on SPI storage
- `emerge-<board> sys-boot/chromeos-bootimage`
  - depends on `coreboot`
    - `/build/$BOARD/firmware/$DEVICE/coreboot.rom`
    - a CBFS archive
    - no payload
  - depends on `depthcharge`
    - `/build/$BOARD/firmware/$BOARD/depthcharge/depthcharge.elf`
    - a coreboot payload
  - it generates
    - `/build/$BOARD/firmware/image-$DEVICE.bin`
    - `/build/$BOARD/firmware/image-$DEVICE.serial.bin` for debug
    - others
- `cbfstool`
  - the bootimage is a CBFS archive
    - repacked from `coreboot.rom` which is also a CBFS archive
  - to see the contents,
    - `cbfstool image-<board>.bin layout`
    - `cbfstool image-<board>.bin print -r COREBOOT`
    - `cbfstool image-<board>.bin print -r RW_LEGACY`
- depthcharge
  - depthcharge is a payload of coreboot
    - it links to `libpayload`
    - it links to vboot
      - `vboot_reference`
  - coreboot loads depthcharge payload which boots cros
    - it can be hacked to load SeaBIOS or TianoCore to boot linux
  - the entry point is `main` in `src/vboot/main.c`
    - it is actually `_entry` (in libpayload) according to
      `depthcharge.ldscript.S`
    - in libpayload, `_entry` jumps to `_init` which calls `start_main` which
      calls `main`
- ebuilds
  - `sys-boot/coreboot` builds coreboot and atf binaries (not the final fw),
    mainly from
    - `third_party/coreboot`
    - `third_party/arm-trusted-firmware`
    - `platform/vboot_reference`
  - `sys-boot/libpayload` builds static libraries and headers from
    - `third_party/coreboot`
    - `platform/vboot_reference`
  - `sys-boot/depthcharge` builds depthcharge binary from
    - `platform/depthcharge`
    - `third_party/coreboot`
    - `platform/vboot_reference`
    - it also links to libpaylaod
  - `sys-boot/chromeos-bootimage` builds the final ap fw by packing the
    various binaries
- it is best to use firmware branch, such as `firmware-rex-15709.B`, for
  `overlays/chromiumos-overlay`
  - this will use the correct commits for various ebuilds
  - workon ebuilds
    - `sys-boot/coreboot`
    - `sys-boot/libpayload`
    - `sys-boot/depthcharge`
    - `sys-boot/chromeos-bootimage`
  - workon repos should use the fw branch
    - `third_party/coreboot`
    - `third_party/arm-trusted-firmware` (if arm)
    - `platform/vboot_reference`
    - `platform/depthcharge`
  - if android, use `firmware-android-*`

## Using depthcharge

- Keyboard shortcuts
  - need to use AP console if physical keyboard does not work in coreboot
  - shutdown the device: hold power key until the device shutdown
  - enter recovery mode: hold ESC and F3/Refresh, then press power button to boot
  - enter developer mode: while in recovery mode, press Ctrl-D
  - boot from usb: at developer mode warning or red screen, press Ctrl-U
    - ctrl-D to boot from disk/ufs
    - ctrl-N to boot from network
    - ctrl-U to boot from USB
      - require `crossystem dev_boot_usb=1` or `enable_dev_usb_boot` in shell first
      - USB stick holds a live image
      - run `chromeos-install` to install to disk/ufs
- behavior partly controlled by GBB flags
  - `set_gbb_flags.sh` or
  - `futility gbb -g --flags <bios>` to get the GBB flag in a bios
- `crossystem` reads/writes system properties from
  - nv storage
  - `/sys/devices/platform/chromeos_acpi/VDAT` (x86)
  - `/proc/device-tree/firmware/chromeos/vboot-shared-data` (ARM)

## depthcharge internals

- the entrypoint is `main` defined in `src/vboot/main.c`
- `vboot_select_and_boot_kernel` starts the booting process
  - `vb2api_kernel_phase1`
  - `vb2api_kernel_phase2`
  - `vboot_select_and_load_kernel`
  - `vb2api_kernel_finalize`
  - `vboot_boot_kernel`
- entering developer mode in depthcharge
  - if recovery, `vboot_select_and_load_kernel` shows `recovery_select_screen`
  - followed by ctrl-d, `ui_manual_recovery_action` shows
    `recovery_to_dev_screen`
  - `recovery_to_dev_finalize` calls `vb2api_enable_developer_mode` to set
    `VB2_SECDATA_FIRMWARE_FLAG_DEV_MODE` and reboot
  - because `VB2_SECDATA_FIRMWARE_FLAG_DEV_MODE` is set, these 3 bits are also
    set
    - `VB2_SD_FLAG_DEV_MODE_ENABLED`
    - `VB2_CONTEXT_DEVELOPER_MODE`
    - `VB2_SECDATA_FIRMWARE_FLAG_LAST_BOOT_DEVELOPER`
  - because of `VB2_CONTEXT_DEVELOPER_MODE`, the boot mode is set to
    `VB2_BOOT_MODE_DEVELOPER`
  - because of `VB2_BOOT_MODE_DEVELOPER`, `vboot_select_and_load_kernel` shows
    `developer_mode_screen`
  - `ui_developer_mode_boot_internal_action` calls `vboot_load_kernel` to load
    the kernel from the internal disk
    - it calls `vb2api_load_kernel` from vboot to load the kernel image
    - internally,
      - `GptNextKernelEntry` picks the kernel with the highest priority
        - it only considers partitions with `S1` or `Tn` with `n>0`
      - `GptUpdateKernelEntry` updates the gpt flags
        - `GPT_UPDATE_ENTRY_TRY` decrements `T` if `S0`
          - if `T` reaches 0, set `P0` as well
        - `GPT_UPDATE_ENTRY_BAD` sets `P0,T0` if `S0`
        - `GPT_UPDATE_ENTRY_ACTIVE` sets `S1,P2,T0`
        - `GPT_UPDATE_ENTRY_INVALID` sets `S0,P0,T0`
  - `vboot_boot_kernel` boots the loaded kernel
    - `fill_boot_info` fills in the boot info including the cmdline
    - `commandline_subst` replaces the cmdline
      - `cros_secure` is prepended to indicate the bootloader is depthcharge
      - `%U` is replaced by kernel partition UUID
    - `crossystem_setup` updates acpi for use by the kernel
      - because of `VB2_CONTEXT_DEVELOPER_MODE`, `vb2api_export_vbsd` exports
        `VBSD_BOOT_DEV_SWITCH_ON` in acpi vdat table
    - `boot` jumps to the kernel
- entering developer mode in userspace
  - upstart `startup` executes `chromeos_startup` defined in platform2
  - `ChromeosStartup::Run`
    - `Platform::InDevMode` calls `VbGetSystemPropertyInt("cros_debug")` to
      see if we are in dev mode yet
      - this looks for `cros_debug` in `/proc/cmdline`
      - <https://chromium.googlesource.com/chromiumos/platform/vboot_reference/+/refs/heads/main/host/lib/crossystem.c>
  - `ChromeosStartup::CheckForStatefulWipe`
    - `ChromeosStartup::IsDevToVerifiedModeTransition`
      - `VbGetSystemPropertyInt("devsw_boot")` checks for
        `VBSD_BOOT_DEV_SWITCH_ON` in acpi vdat table
      - `VbGetSystemPropertyString("mainfw_type")` must not be `recovery`
    - `ChromeosStartup::DevIsDebugBuild` checks for
      `VbGetSystemPropertyInt("debug_build")`
    - `Platform::Clobber`
      - it invokes `/sbin/chromeos-boot-alert` with `enter_dev`
        - this shows `Preparing system for Developer Mode...` and waits for
          30s
      - it invokes `/sbin/clobber-log` with `Enter developer mode`
        - this logs to `/mnt/stateful_partition/unencrypted/clobber.log`
      - it invokes `/sbin/clobber-state` with `keepimg`
        - this wipes and re-creates stateful
        - this touches `/mnt/stateful_partition/.developer_mode`
        - this reboots the device
  - crossystem variables
    - `devsw_boot` means devloper mode in depthcharge
      - it is true after entering developer mode 
    - `cros_debug` means developer mode in userspace
      - it is true when `/proc/cmdline` has `cros_debug`
      - dev/test images are built with `--force_developer_mode` , which sets
        `cros_debug` in kernel cmdline

## depthcharge altfw

- when building chromeos-bootimage,
  - `USE=seabios` to include seabios
  - `USE=tianocore` to include UEFI
  - `USE=u-boot` to include u-boot
- `cbfstool /build/$BOARD/firmware/image-$DEVICE.bin print -r RW_LEGACY` to
  see the included payloads
- <https://chromium.googlesource.com/chromiumos/docs/+/refs/heads/master/developer_mode.md#alt-firmware>
  - `flashrom -r /tmp/bios.bin`
  - `cbfstool /tmp/bios.bin print -r RW_LEGACY`
  - `cbfstool /tmp/bios.bin add-payload -r RW_LEGACY -c lzma -n <your bootloader name> -f <path/to/your/bootloader.elf>`
  - `cbfstool /tmp/bios.bin extract -r RW_LEGACY -n altfw/list -f /tmp/altfw.txt`
  - `cbfstool /tmp/bios.bin remove -r RW_LEGACY -n altfw/list`
  - `cbfstool /tmp/bios.bin add -r RW_LEGACY -n altfw/list -f /tmp/altfw.txt -t raw`
  - `cbfstool /tmp/bios.bin remove -r RW_LEGACY -n cros_allow_auto_update`
  - `flashrom -w /tmp/bios.bin -i RW_LEGACY`
  - `crossystem dev_boot_legacy=1`
  - `Ctrl-L` in boot prompt
