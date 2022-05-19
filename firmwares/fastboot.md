Android fastboot
================

## Overview

- <https://source.android.com/devices/bootloader>
- the only open-source bootloader is a fork of lk
  - <https://source.codeaurora.org/quic/la/kernel/lk/>

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
