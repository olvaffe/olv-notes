Chrome OS GSC and EC
====================

## HW Overview

- <https://2018.osfc.io/uploads/talk/paper/7/gsc_copy.pdf>
- in addition to CPU, known as AP, there are also GSC (Google Security Chip)
  and EC
- GSC pins
  - power related
    - power button
    - AC present
    - battery cutoff
    - EC reset
    - AP reset
  - flash related
    - WP to AP & EC flashes
    - SPI to API & EC flashes
  - tpm
    - SPI to AP
  - debug related
    - usb-c (for CCD)
    - EC uart
    - AP uart
- in other words,
  - GSC can flash its own flash, EC flash, and AP flash
  - GSC can connect to EC uart and AP uart
  - devs can access EC using a special usb-c SuzyQ cable
  - devs can also access EC from AP via TPM
    - thus devs can update GSC, EC, and AP firmwares on DUT

## Firmware Overview

- GSC firmware
  - GSC has a bootrom and a 512KB flash
    - bootrom is read-only
    - the 512KB flash is divided into two (16 KB RO, 228KB RW, 12KB NVMEM)
  - Cr50, the name of the firmware, is built from
    <ttps://chromium.googlesource.com/chromiumos/platform/ec>
  - flash from dut
    - `gsctool -a -f` to get the RW version
      - 0.3.x for MP
      - 0.4.x for PrePVT
    - `gsctool -a /opt/google/cr50/firmware/cr50.bin.prod` to flash
  - flash from host
    - `gsctool` to flash over usb
    - `cr50-rescue` to flash over uart
- EC firmware
  - some EC has built-in flash and some has external flash
  - the firmware is also built from
    <ttps://chromium.googlesource.com/chromiumos/platform/ec>
  - flash from dut
    - `ectool version` to get current version
    - `flashrom -p ec -w <ec.bin>` to flash
    - `chromeos-firmwareupdate`
  - flash from host
    - `flash_ec`
- AP firmware
  - <bootloader.md>
  - flash from dut
    - `crossystem fwid` to get current version
    - `flashrom -p host -w <image>` to flash
    - `chromeos-firmwareupdate`
  - flash from host
    - `flashrom -p raiden_debug_spi:target=AP -w $IMAGE`
    - `cros ap flash`
- kernel and userspace
  - `./update_kernel.sh`
  - `cros flash`
