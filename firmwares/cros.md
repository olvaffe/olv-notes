Chrome OS Firmwares
===================

## Flash Firmwares

- NEVER FLASH ANY FIRMWARE WITHOUT HAVING SUZYQ TO UNBRICK
- on a device that has been setup, `chromeos-firmwareupdate -m recovery`
  should work
- on a new device,
  - the running firmwares are signed by prod keys
  - our firmwares from `chromeos-firmwareupdate` on the latest build are
    likely unsigned
  - without CCD open, we can only flash RW sections and the RO sections will
    refuse to boot
  - after CCD open, they might refuse to boot as well
  - have suzyq ready and be prepared to flash from host
- getting firmwares
  - one should use known-good versions for CR50, EC and AP instead
    - in my experience, the latest EC and AP firmwares do not boot
    - the recommended versions from CPFE might be oudated or might not boot
      - <https://chromium.googlesource.com/chromiumos/docs/+/HEAD/archive_mirrors.md#private-mirrors>
  - best is to flash the latest build and copy these firmwares to host
    - `/usr/sbin/chromeos-firmwareupdate`
    - `/opt/google/cr50/firmware`
  - but if your device does not boot,
    - for cr50, `emerge chromeos-cr50` in chroot
    - for EC/AP, each board provides its `virtual/chromeos-firmware` that
      depends on `chromeos-base/chromeos-firmware-$BOARD`
      - `chromeos-base/chromeos-firmware-$BOARD/files/srcuris` are uris for
      	the EC/AP firmwares
- to flash GSC, use `gsctool` on dut or host
  - `gsctool -f` to get the running firmware version
  - `gsctool -b <cr50-firmware>` to check the firmware version
  - `gsctool <cr50-firmware>`, to update the RW firmware
  - on dut,
    - `gsctool -a -f` to get the running version
    - `gsctool -a -b /opt/google/cr50/firmware/cr50.bin.prod` to get the image
      version
      - `cr50.bin.prod` is for MP devices and has RW version `0.5.*`
      - `cr50.bin.prepvt` is for pre-MP devices and has RW version `0.6.*`
      - try to match the version of the running version
    - `gsctool -a /opt/google/cr50/firmware/cr50.bin.prod` to flash
  - on host,
    - `emerge chromeos-cr50-dev chromeos-cr50` and use `gsctool` to flash over
      suzyqable
    - use `cr50-rescue` to flash over uart as a final resort
- to ccd open, use `gsctool` on dut or host
  - this is mainly to disable wp
  - see `CCD` section below
- to flash EC, use `flash_ec` script on host
  - `emerge ec-devutils` to get `flash_ec`
  - requires working servod
  - `flash_ec --board=<boardname> [--zephyr] [--image=<path/to/ec.bin>]`
    - it might need `npcx_monitor.bin` from `emerge-$board chromeos-base/chromeos-ec`
  - one can also flash EC on dut
    - `futility`
    - `flashrom -p ec -r <backup.bin>`
    - `flashrom -p ec -w <path-to/ec.bin>`
- to flash AP on host,
  - `futility update --servo -i image-<board>.bin`
  - or, `cros ap flash -i image-<board>.bin`
  - or, `flashrom -p raiden_debug_spi:target=AP -w $IMAGE`
  - it takes multiple (5-10) minutes to run!  Be patient.
- to flash AP on dut,
  - `crossystem fwid` to get current version
  - `futility update -i image-<board>.bin` to flash
  - or, `flashrom -p host -w <image>`
  - might need to disable write-protection, `flashrom --wp-disable`

## Google Security Chip (GSC), currently H1

- <https://2018.osfc.io/uploads/talk/paper/7/gsc_copy.pdf>
- ARM SC300 core, 8kB boot ROM, 64kB SRAM, 512kB flash
- USB, I2C, SPI, UART, GPIO, etc.
- in addition to CPU, known as AP, there are also GSC (Google Security Chip)
  and EC
  - GSC does not replace EC
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
- Cr50, firmware on GSC
  - Cr50 runs on GSC and implements serials, TPM2, CCD, and U2F
    - <https://chromium.googlesource.com/chromiumos/platform/ec>
    - `cr50_stab` branch
    - `make BOARD=cr50`
    - ebuild is `chromeos-base/chromeos-cr50`
  - Cr50 does not reboot (unless the battery dies)
  - GSC has a bootrom and a 512KB flash
    - bootrom is read-only
    - the 512KB flash is divided into two (16 KB RO, 228KB RW, 12KB NVMEM)
      - two copies of Cr50 for A/B update
    - coming out of reboot, GSC boots to RO first;  RO then boots to RW.

## EC

- some EC has built-in flash and some has external flash
- EC firmware is now zephyr-based
  - <https://chromium.googlesource.com/chromiumos/platform/ec/+/HEAD/docs/zephyr/README.md>
  - `zmake <board>`
  - `build/zephyr/<board>/output/zephyr.bin`
- previously, it was based on
  - <https://chromium.googlesource.com/chromiumos/platform/ec>
  - `make BOARD=<board>`
    - the binary will be at `build/<board>/ec.bin`
    - or, `emerge-<board> chromeos-base/chromeos-ec`
  - source code directory structure
    - A board under `board/` decides the baseboard
    - A baseboard under `baseboard/` decides the controller
    - A controller under `chip/` decides the CPU under `core/`
  - To get ec.bin,
    - `$(out)/%.elf: $(out)/%.lds $(objs)`
      - use ec.lds to link
    - `$(out)/%.flat: $(out)/%.elf $(out)/%.smap $(build-utils)`
      - create EC binary image
    - `$(out)/$(PROJECT).obj: common/firmware_image.S $(out)/firmware_image.lds $(flat-y)`
      - create firmware image that combines several binary images
    - `$(out)/%.bin: $(out)/%.obj`
      - create firmware binary

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

## Suzy-Q

- Suzy-Q allows a host device to talk to DUT's Cr50/EC/AP
  - It is a USB-A to USB-C cable
  - The USB-C end is non-standard
  - It needs to be connect to a specific port of DUT and in a specific
    orientation to work.
  - Once connected, several USB devices show up to the host
    - 18d1:501f Google Inc. SuzyQable
    - 18d1:5014 Google Inc. Cr50
      - this shows up only when the USB-C end is connected correctly
- Cr50 makes three TTY endpoints available to the host
  - requires `usbserial` in the host kernel; /dev/ttyUSBx
  - first TTY is Cr50 console
    - `minicom -D /dev/ttyUSB0` for Cr50 console
    - `version` to get version
  - second TTY is AP console
    - `minicom -D /dev/ttyUSB1` for AP console
    - empty unless AP firmware (BIOS) is rebuilt to output debug messages
  - third TTY is EC console
    - `minicom -D /dev/ttyUSB2` for EC console
    - read-only unless CCD is opened
- `servod`
  - `emerge hdctools`, hardware debug and control tools
    - `servod -b $BOARD` to start the servo daemon
    - `dut-control` to talk to `servod`
      - `dut-control cr50_uart_pty` for cr50 uart
      - `dut-control ec_uart_pty` for ec uart
      - `dut-control cpu_uart_pty` for cpu uart
        - non-dev bios does not output to cpu uart
        - kernel might output to cpu uart
  - `servod` supports a wide range of interfaces
    - cr50 ccd: 18d1:5014
      - GSC's cr50 firmware implements ccd and servo protocol
      - requires CCD open first
    - servo micro: 18d1:501a
    - servo v1: 18d1:5001
    - servo v2: 18d1:5002
    - servo v3: 18d1:5004
    - servo v4: 18d1:501b
      - Servo v4 itself has three USB endpoints that are TTY devices
      - don't get confused with the cr50/ap/ec consoles
      - the firmware is also built from
        <https://chromium.googlesource.com/chromiumos/platform/ec>
        - `make BOARD=servo_v4`
      - `sudo servo_updater -b servo_v4` to update the firmware
    - servo v4.1: 18d1:520d
    - c2d2: 18d1:5041
    - more
- long time ago, there was no GSC/Cr50 but only EC
  - to debug, servo v1/v2/v3 must be connected to a special debug head in DUT
  - servo v1/v2/v3 appears as a USB device on the host machine
  - the dongle executes commands from servod
  - to flash EC or AP from host,
    - `flashrom -V -p raiden_debug_spi:target=AP -w image-<board>.bin`
    - `flashrom -V -p raiden_debug_spi:target=EC -w ec.bin`
- then, there was GSC/Cr50
  - to debug, servo v4 or suzyqable must be connected to a special usb-c port
    in DUT in a specific orientation
  - Cr50 appears as a USB device on the host machine
  - CCD must be opened for Cr50
    - `gsctool -o`
  - Cr50 executes commands from servod
  - `flashrom` still works

## CCD

- <https://chromium.googlesource.com/chromiumos/platform/ec/+/cr50_stab/docs/case_closed_debugging_cr50.md>
- on DUT or host with working `gsctool`
  - `gsctool -a -o`
  - press power key occasionally for 5 minutes
  - `minicom -D /dev/ttyUSB0`
    - we can also do `ccd open` here instead of `gsctool -o`
  - `ccd reset factory`
  - `ccd testlab enable`
  - press power key several times
- to confirm
  - `reboot`
  - `ccd testlab` should print `CCD test lab mode enabled`
  - `ccd` should have `State: Opened`
  - `wp` should have `force disabled`

## `gsctool`

- gsctool is a helper tool to update Cr50
  - `extra/usb_updater/gsctool.c` of EC `cr50_stab` branch
  - `emerge chromeos-cr50-dev`, to get gsctool
- On host, `gsctool` uses Cr50 usb device (18d1:5014)
  - `gsctool -d 18d1:5014`
  - DUT needs to be connected via SuzyQ
- On DUT, `gsctool` ueses TPM
  - `gsctool -s` to use `/dev/tpm0` directly
- build firmware
  - usually unncessary as it is already built and installed on all devices
  - otherwise, `emerge-<board> chromeos-cr50`, to build the latest RW images
    for DUT
    - prod image is for MP devices
    - pre-pvt image is for pre-pvt devices and developers

## `ectool`

- to communicate with EC
- kernel exposes `/dev/cros_ec` on DUT
  - `CONFIG_CROS_EC`
  - `CONFIG_MFD_CROS_EC_DEV`
- `ectool` talk to EC via `/dev/cros_ec`
  - <https://chromium.googlesource.com/chromiumos/platform/ec/+/master/util/ectool.c>
- useful commands
  - `ectool version` prints EC version
  - `ectool hello` checks basic EC communication
  - `ectool usbpdpower` prints USB PD info
  - `ectool temps all` prints temperatures
  - `ectool switches` prints switch info
  - `ectool pwmgetkblight` prints keyboard backlight percentage
  - `ectool flashinfo` prints flash info
  - `ectool flashspiinfo` prints flash spi info
  - `ectool chipinfo` prints chip info
  - `ectool battery` prints battery info
- EC console is accessible from host
  - requires a debug table
  - read-only until CCD is opened

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
