## coreboot

 - coreboot lives on SPI storage
   - use `futility update` or `flashrom` to flash a new image
   - might need to disable write-protection, `flashrom --wp-disable`
   - to backup, `sudo flashrom -V -p raiden_debug_spi:target=AP -r old.bin`
   - to update, `sudo flashrom -V -p raiden_debug_spi:target=AP -w new.bin`
   - it takes multiple (5-10) minutes to run!  Be patient.
 - coreboot loads depthcharge payload which boots cros
   - it can be hacked to load SeaBIOS or TianoCore to boot linux

## depthcharge

- Keyboard shortcuts
  - need to use AP console if physical keyboard does not work in coreboot
  - shutdown the device: hold power key until the device shutdown
  - enter recovery mode: hold ESC and F3/Refresh, then press power button to boot
  - enter developer mode: while in recovery mode, press Ctrl-D
  - boot from usb: at developer mode warning or red screen, press Ctrl-U
    - ctrl-D to boot from disk/ufs
    - ctrl-N to boot from network
    - ctrl-U to boot from USB
      - require "crossystem dev_boot_usb=1" or "enable_dev_usb_boot" in shell first
      - USB stick holds a live image
      - run "chromeos-install" to install to disk/ufs
- behavior partly controlled by GBB flags
  - set_gbb_flags.sh or
  - "futility gbb -g --flags <bios>" to get the GBB flag in a bios
- crossystem reads/writes system properties from
  - nv storage
  - /sys/devices/platform/chromeos_acpi/VDAT (x86)
  - /proc/device-tree/firmware/chromeos/vboot-shared-data (ARM)
- the entry point is `main` in `src/vboot/main.c`
  - it is actually `_entry` (in libpayload) according to
    `depthcharge.ldscript.S`
  - in libpayload, `_entry` jumps to `_init` which calls `start_main` which
    calls `main`
