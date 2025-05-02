Chrome OS Firmware
==================

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
    - looks like the device exits developer mode after CCD open
    - re-entering fixes it
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
- flash on host over servo
  - gsc
    - `gsctool -d 18d1:504a -f` to get running version
    - `gsctool -b <img>` to get image version
    - `gsctool -d 18d1:504a <img>` to flash
  - ec
    - `flash_ec --board rauru --image <ec-img> --zephyr` to flash
  - ap
    - `futility update --servo -m factory -i <ap-img>` to flash
      - this shows the image version and the running version
      - ap ro includes a section called gbb
        - `futility gbb --get --flags <ap-img>` to show the flags
        - 0x1: reduces dev screen delay from 30s to 2s
        - 0x8: forces dev switch on
        - 0x10: always allow usb boot
        - 0x20: disable fw rollback
        - 0x200: disable ec sw sync
          - modern ap img includes ec img, and will update ec rw on next boot
          - the mechanism is called ec sw sync
          - this flag is useful when ec sw sync fails persistently
            - persistently getting updating critical firmware and entering
              recovery mode
- flash on dut
  - gsc
    - `gsctool -a -f` to get running version
    - `gsctool -b <img>` to get image version
    - `gsctool -a <img>` to flash
  - ec and ap
    - `chromeos-firmwareupdate -m factory`

## GSC (Google Security Chip)

- titan
  - titan is a family of silicon root of trust (secure microcontrollers)
    developed by google
  - they are used in different products
    - datacenter servers
    - chromebooks
    - pixels
    - security keys
  - opentitan is an open-source design based on titan
  - tock os is an open-source os with titan and opentitan support
  - what cros calls gsc is titan
- titan H1
  - it is an older soc
  - <https://2018.osfc.io/uploads/talk/paper/7/gsc_copy.pdf>
    - ARM SC300 core, 8kB boot ROM, 64kB SRAM, 512kB flash
    - USB, I2C, SPI, UART, GPIO, etc.
  - pins
    - power related
      - power/refresh/back keys
      - AC present
      - battery cutoff
      - EC reset
      - AP reset
    - flash related
      - WP to AP & EC flashes
      - SPI to AP & EC flashes
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
- cr50 / ti50
  - cr50 is the older firmware that runs on titan H1
  - it is a fork of <https://chromium.googlesource.com/chromiumos/platform/ec>
    for the apps (tpm, ccd, u2f, serial, etc.) and kernel
    - `cr50_stab` branch
    - `make BOARD=cr50`
    - ebuild is `chromeos-base/chromeos-cr50`
  - H1 has a bootrom and a 512KB flash
    - bootrom is read-only
    - the 512KB flash is divided into two (16 KB RO, 228KB RW, 12KB NVMEM)
      - two copies of Cr50 for A/B update
    - coming out of reset, bootrom boots to RO first;  RO then boots to RW.
  - the tpm app exposes a tpm device to the dut
  - the ccd app exposes a usb device with id `18d1:5014` when connected to the
    host with servo
    - the usb device provides many functions
    - up to 4 serial ports
      - 0 for cr50 serial
      - 1 for ap serial
      - 2 for ec serial
      - 3 for fpmcu serial
    - `raiden_debug_spi` flash programmer
      - because gsc connects to both ec and ap flashes, `target={ec,ap}`
        should be specified as well
      - use `gsctool` to flash gsc firmware instead
- ti50
  - ti50 is the newer firmware that runs on titan D2
  - it consists of a fork of cros ec for the apps and a fork of tock os for
    the kernel
  - the ccd app exposes a usb device with id `18d1:504a`

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
- Cr50 also implements servo and ccd protocols
  - this allows `servod` to work over Cr50
  - requires CCD open first

## Servo

- <https://chromium.googlesource.com/chromiumos/third_party/hdctools/+/HEAD/docs/servo_v4p1.md>
  - servo v4.1 has usb id `18d1:520d`
  - it has three USB endpoints that are TTY devices
    - don't get confused with the cr50/ap/ec consoles
  - ports
    - the host should connect to the `HOST` port
      - if the host cannot supply enough power, a charger can connect to the
        `SERVO POWER` port
    - a charger should connect to `DUT POWER` port to charge the DUT
    - the capative USB-C should connect to DUT
  - `sudo servo_updater -b servo_v4p1` to update the firmware
    - the firmware is built from
      <https://chromium.googlesource.com/chromiumos/platform/ec>
    - `make BOARD=servo_v4p1`
  - if servo v4.1 keeps rebooting and the usb device keeps reconnecting, the
    usb firmware needs to be updated first
    - `sudo systemctl stop fwupd`
      - this should work around the rebooting issue
    - `sudo fwupdtool get-devices`
      - this should show a `Servo Dock` device
        - if no, and if you get `no CFI device found` instead, make sure
          fwupd is at least 1.8.9
      - if the firmware version is below 64.17, it is too old
    - `sudo fwupdtool update`
      - or visit
        <https://fwupd.org/lvfs/devices/tw.com.genesyslogic.gl3590.firmware>
      - `sudo fwupdtool install be2c9146ff4cfac5d647376c39ce0b78151e9f1a785a287e93ac3968aff2ed50-GenesysLogic_GL3590_64.17.cab`
- `servod` supports a wide range of interfaces
  - servo v4.1: 18d1:520d
  - cr50 ccd: 18d1:5014
    - GSC's cr50 firmware implements ccd and servo protocol
    - requires CCD open first
  - others
    - servo micro: 18d1:501a
    - servo v1: 18d1:5001
    - servo v2: 18d1:5002
    - servo v3: 18d1:5004
    - servo v4: 18d1:501b
    - c2d2: 18d1:5041
    - more
- `servod` usage
  - `emerge hdctools`, hardware debug and control tools
  - `servod -b $BOARD` to start the servo daemon
  - `dut-control` to talk to `servod`
    - `dut-control cr50_uart_pty` for cr50 uart
    - `dut-control ec_uart_pty` for ec uart
    - `dut-control cpu_uart_pty` for cpu uart
      - non-dev bios does not output to cpu uart
      - kernel might output to cpu uart
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

- <https://chromium.googlesource.com/chromiumos/platform/ec/+/cr50_stab/docs/case_closed_debugging_gsc.md>
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
    - if not, `ccd open` to open it
    - testlab allows it to be opened anytime
  - `wp` should have `force disabled`
- if the ap firmware says "something went wrong", try entering developer mode
  again

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

## Command Lines

- `dut-control lid_open:no` closes the lid
  - `yes` to open
