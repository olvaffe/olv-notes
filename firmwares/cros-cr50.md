Chrome OS Cr50
==============

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

## Cr50, firmware on GSC

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
- communication
  - A host machine can talk to Cr50, when a SuzyQ cable is used to connect
    host and DUT
  - The AP (the firmware/os runs on the DUT AP CPU) can also talk to Cr50 via
    TPM

## Flash Cr50

- gsctool is a helper tool to update Cr50
  - `extra/usb_updater/gsctool.c` of EC `cr50_stab` branch
  - `emerge ec-utils`, to get gsctool
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
- NEVER FLASH ANY FIRMWARE WITHOUT HAVING SUZYQ TO UNBRICK
- using gsctool
  - `gsctool -f` to get the running firmware version
  - `gsctool -b <cr50-firmware>` to check the firmware version
  - `gsctool <cr50-firmware>`, to update the RW firmware
- in short, to flash from dut
  - `gsctool -a -f` to get the running version
  - `gsctool -a -b /opt/google/cr50/firmware/cr50.bin.prod` to get the image
    version
  - `gsctool -a /opt/google/cr50/firmware/cr50.bin.prod` to flash
- to flash from host
  - `gsctool` to flash over usb
  - `cr50-rescue` to flash over uart

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

## Servo

- `emerge hdctools`, hardware debug and control tools
  - `servod` to start the servo daemon
  - `dut-control` to talk to `servod`
- `servod` supports a wide range of interfaces
  - cr50 ccd: 18d1:5014
    - cr50 implements ccd and servo protocol
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
- long time ago, there was no Cr50 but only EC
  - to debug, servo v1/v2/v3 must be connected to a special debug head in DUT
  - servo v1/v2/v3 appears as a USB device on the host machine
  - the dongle executes commands from servod
  - to flash EC or AP from host,
    - `flashrom -V -p raiden_debug_spi:target=AP -w image-<board>.bin`
    - `flashrom -V -p raiden_debug_spi:target=EC -w ec.bin`
- then, there was Cr50
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
  - `gsctool -o`
  - press power key occasionally for 5 minutes
  - `minicom -D /dev/ttyUSB0`
  - `ccd reset factory`
  - `ccd testlab enable`
  - press power key several times
