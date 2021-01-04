EC
==

## Directory Structure

- A board under `board/` decides the baseboard
- A baseboard under `baseboard/` decides the controller
- A controller under `chip/` decides the CPU under `core/`

## Build System

- To get ec.bin,
  - `$(out)/%.elf: $(out)/%.lds $(objs)`
    - use ec.lds to link
  - `$(out)/%.flat: $(out)/%.elf $(out)/%.smap $(build-utils)`
    - create EC binary image
  - `$(out)/$(PROJECT).obj: common/firmware_image.S $(out)/firmware_image.lds $(flat-y)`
    - create firmware image that combines several binary images
  - `$(out)/%.bin: $(out)/%.obj`
    - create firmware binary

## `ectool`

- kernel has `/dev/cros_ec` on DUT
  - `CONFIG_CROS_EC`
  - `CONFIG_MFD_CROS_EC_DEV`
- `ectool` talk to EC via `/dev/cros_ec`
  - <https://chromium.googlesource.com/chromiumos/platform/ec/+/master/util/ectool.c>
- useful commands
  - `ectool hello` checks basic EC communication
  - `ectool version` prints EC version
  - `ectool usbpdpower` prints USB PD info
  - `ectool temps all` prints temperatures
  - `ectool switches` prints switch info
  - `ectool pwmgetkblight` prints keyboard backlight percentage
  - `ectool flashinfo` prints flash info
  - `ectool flashspiinfo` prints flash spi info
  - `ectool chipinfo` prints chip info
  - `ectool battery` prints battery info

## Update EC firmware

- `flashrom` on DUT
  - `flashrom -p ec -r <backup.bin>`
  - `flashrom -p ec -w <path-to/ec.bin>`
- `flash_ec` on host
  - requires servo

## Cr50, firmware on H1 secure microcontroller

- Google Security Chip H1
  - <https://2018.osfc.io/uploads/talk/paper/7/gsc_copy.pdf>
  - <https://chromium.googlesource.com/chromiumos/platform/ec/+/master/README.md>
  - ARM SC300 core, 8kB boot ROM, 64kB SRAM, 512kB flash
  - USB, I2C, SPI, UART, GPIO, etc.
  - flash contains two copies of Cr50 for redundency
  - each copy is devided into RO and RW sections
  - coming out of reboot, it boots to RO first;  RO then boots to RW.
- Cr50 runs on H1 and implements TPM2, CCD, and U2F
  - the Cr50 firmware is also built from Chromium OS EC
    - <https://chromium.googlesource.com/chromiumos/platform/ec>
    - `cr50_stab` branch
    - `BOARD=cr50`
  - Cr50 does not reboot (unless the battery dies)
- A host device can talk to Cr50, when a SuzyQ cable is used to connect them
- The AP (the firmware/os runs on the AP CPU) can also talk to Cr50 via TPM

## `gsctool`

- gsctool is a helper tool to update Cr50
  - `extra/usb_updater/gsctool.c` of EC `cr50_stab` branch
  - `emerge ec-utils`, to get gsctool
- On host, `gsctool` uses Cr50 usb device
  - `gsctool -d 18d1:5014`
  - DUT needs to be connected via a debug cable
- On DUT, `gsctool` ueses TPM
  - `gsctool -s` to use `/dev/tpm0` directly
  - `gsctool -t` to talk to `trunksd` which alreay opens `/dev/tpm0`
- to update Cr50,
  - `emerge-<board> chromeos-cr50`, to get the latest RW images for DUT
    - prod image is for MP devices
    - pre-pvt image is for pre-pvt devices and developers
  - `gsctool <cr50-firmware>`, to update the RW firmware
  - `gsctool -f` to get the running firmware version
  - `gsctool -b <cr50-firmware>` to check the firmware version

## Suzy-Q

- Suzy-Q allows a host device to talk to DUT's EC/Cr50/AP
  - It is a USB-A to USB-C cable.  The USB-C end is non-standard.  It needs
    to be connect to a specific port of DUT and in a specific orientation to
    work.
  - Once connected, several USB devices show up to the host
    - 18d1:501f Google Inc. SuzyQable
    - 18d1:5014 Google Inc. Cr50
      - this shows up only when the cable is connected correctly
- Cr50 makes three TTY endpoints available to the host
  - requires usbserial in the host kernel; /dev/ttyUSBx
  - first TTY is Cr50 console
  - second TTY is AP console
  - third TTY is EC console
- `minicom -D /dev/ttyUSB0` for Cr50 console
  - `version` to get version
- `minicom -D /dev/ttyUSB1` for AP console
- `minicom -D /dev/ttyUSB2` for EC console

## Servo

- Servo
  - usb id 18d1:501b is Servo v4
  - Servo v4 itself has three USB endpoints that are TTY devices
    - don't get confused with the cr50/ap/ec consoles
- `emerge hdctools`, hardware debug and control tools
  - `sudo servo_updater -b servo_v4` to update the firmware
  - `servod` to start the servo daemon
  - `dut-control`
- to update EC,
  - `flash_ec --board=<boardname> [--image=<path/to/ec.bin>]`
