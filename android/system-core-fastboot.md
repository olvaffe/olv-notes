Android fastboot
================

## Overview

- <https://source.android.com/devices/bootloader>
- depthcharge is a bootloader with fastboot support
  - <https://chromium.googlesource.com/chromiumos/platform/depthcharge/+/refs/heads/main/src/fastboot/>
- `adb reboot bootloader` reboots to bootloader
  - bootloader supports fastboot protocol, but does not understand dynamic
    partitions
- `adb reboot fastboot` reboots to fastbootd if `ro.boot.dynamic_partitions`,
  or falls back to bootloader otherwise
  - fastbootd supports fastboot protocol, and understands dynamic partitions
- bootloader also supports magic key combo to force fastboot handling than
  booting the kernel
  - depthcharge supports `Ctrl-F`

## Lock / Unlock

- On a working device, go to `Settings > System > Developer options` menu
  - enable the `OEM unlocking` option
  - this sets `get_unlock_ability` somewhere to 1 and is persistent across
    reboots and factory resets
- to unlcok the bootloader, `fastboot flashing unlock`
  - this checks `get_unlock_ability`, prompts for confirmation, and wipes the
    device
- <https://android.googlesource.com/platform/packages/apps/Settings/+/master/src/com/android/settings/development/OemUnlockPreferenceController.java>
  - only devices that have `ro.oem_unlock_supported` has the `OEM unlocking`
    option
  - `OEM unlocking` option is grayed out if `enableOemUnlockPreference` returns false
    - it checks that the bootloader is not locked (`ro.boot.flash.locked`)
    - it checks that OEM allows unlock (device-dependent)
  - <https://android.googlesource.com/platform/frameworks/base/+/master/services/core/java/com/android/server/oemlock/OemLockService.java>

## Flash Images

- <https://developers.google.com/android/ota>
  - does not wipe the device
  - no need to unlock the bootloader
  - steps
    - boot into recovery mode
      - `adb reboot recovery`
      - or choose `Recovery` from the bootloader menu
    - choose `Apply update from ADB`
    - `adb sideload ota_file.zip` on the host
  - `payload.bin` consists of blobs that can be extracted to individual files
      - <https://chromium.googlesource.com/aosp/platform/system/update_engine/+/HEAD/README.md#update-payload-file-specification>
      - blobs are filesystem images, etc.
      - google for scripts
- <https://developers.google.com/android/images>
  - steps
    - enable `OEM unlocking`
    - boot into the bootloader, `adb reboot bootloader`
    - unlock the bootloader, `fastboot flashing unlock`
    - unzip and run `flash-all.sh`
  - `flash-all.sh`
    - `fastboot flash bootloader <bootloader.img>`
    - `fastboot flash radio <radio.img>`
      - both images are in qualcomm FBPK format and can be extracted
    - `fastboot -w update <update.zip>`
      - a bunch of Android boot images (created with `mkbootimg`) and sparse
        images (created with `img2simg`)

## Commands

- `fastboot flash` sends `flash:<part>`
- `fastboot getvar foo` sends `getvar:foo`
  - depthcharge has
    - `current-slot:_a`
    - `partition-size:<part>:<size>`
    - `partition-type:<part>:<type>`
    - `has-slot:<part>:<yes-or-no>`
    - many more
- `fastboot reboot` sends `reboot`
  - this manipulates BCB (Bootloader Control Block) in the misc partition
- `fastboot flashing lock|unlock`
- `fastboot set_active` sends `set_active:<slot>`
- `fastboot oem` sends `oem args`
