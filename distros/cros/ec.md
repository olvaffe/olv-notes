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

## Cr50, H1 based security microcontroller

- Google Security Chip H1
  - <https://osfc.io/uploads/talk/paper/7/gsc_copy.pdf>
  - <https://www.chromium.org/chromium-os/ec-development>
  - ARM SC300 core, 8kB boot ROM, 64kB SRAM, 512kB flash
  - USB, I2C, SPI, UART, GPIO, etc.
  - flash contains two copies of Cr50 for redundency
  - each copy is devided into RO and RW sections
  - coming out of reboot, it boots to RO first;  RO then boots to RW.
- Cr50 runs on H1 and implements TPM2, CCD, and U2F
  - the Cr50 firmware is built from Chromium OS EC
  - <https://chromium.googlesource.com/chromiumos/platform/ec>
  - Cr50 does not reboot (unless the battery dies)
  - A host device can talk to Cr50, when a SuzyQ cable is used to connect them
  - The AP (the firmware/os runs on the AP CPU) can also talk to Cr50 via TPM
- Suzy-Q allows a host device to talk to DUT's Cr50
  - It is a USB-A to USB-C cable.  The USB-C end is non-standard.  It needs
    to be connect to a specific port of DUT and in a specific orientation to
    work.
  - Once connected, several USB devices show up to the host
    - 18d1:501f is Suzy-Q itself
      - 18d1:501b is Servo v4
    - 18d1:5014 is Cr50
  - Cr50 makes three TTY endpoints available to the host
    - requires usbserial in the host kernel; /dev/ttyUSBx
    - first TTY is Cr50 console
    - second TTY is AP console
    - third TTY is EC console
  - Servo v4 itself also has three USB endpoints that are TTY devices
- gsctool is a helper tool to talk to Cr50
  - On host, it uses Cr50 console over serial
  - On DUT, it ueses TPM
  - `emerge ec-utils`, to get gsctool
  - `emerge-<board> chromeos-cr50`, to get the latest RW images for DUT
    - prod image is for MP devices
    - pre-pvt image is for pre-pvt devices and developers
  - `gsctool <cr50-firmware>`, to update the RW firmware
  - `gsctool -f`, or type `version` in Cr50 console, to get the RW firmware version
- Servo
  - `emerge hdctools`, hardware debug and control tools
  - `sudo servo_updater -b servo_v4` to update the firmware
  - `servod` to start the servo daemon
  - `dut-control`
- `flash_ec` and `flashrom` are other ways to update Cr50
  - on host: `flash_ec --board=<boardname> [--image=<path/to/ec.bin>]`
  - on DUT: `flashrom -p ec -w <path-to/ec.bin>`
  - untested
