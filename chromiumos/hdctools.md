# Chrome OS hdctools

## `start-servod`

- `start-servod`
  - it checks if docker is up and user belongs to docker group
  - if `development_environment` contents have changed, it rebuilds the
    bootstrap image
    - `docker build -f Dockerfile.bootstrap -t servod-bootstrap ../development_environment`
    - the base is `python:3.11-slim-bookworm`, debian bookworm plus python 3.11
    - `checksum` is the prev checksum to detect content change
  - it starts the bootstrap container to run `start_servod.py`
- `start_servod.py`
  - `parse_args`
    - `channel` defaults to `local`
    - `container_name`, `board`, `model`, `mount`, etc has no default
  - `get_image`
    - because channel defaults to `local`, it looks for `servod:dev` and falls
      back to `us-docker.pkg.dev/chromeos-hw-tools/servod/servod:release`
  - `start_servod`
    - it starts `servod` container to run `start_servod_dev.sh`
      - the args are `--port 9999 --board <board> --model <model>`
  - it exits and the bootstrap container is removed
- `start_servod_dev.sh`
  - it updates usb hub fw on servo if necessary
  - it starts the grpc server
    - `DriverService` talks to the dut
    - `SystemConfig` parses xmls
  - it starts `servod`
    - `ServoService` monitors the grpc server
  - `dut-control` will talk to `servod` which forwards the calls to grpc

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

## Command Lines

- `dut-control lid_open:no` closes the lid
  - `yes` to open
