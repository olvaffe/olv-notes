Chrome OS EC Firmware
=====================

## EC (Embedded Controller)

- EC
  - it is evolved from keyboard controller
  - these peripherals are connected to EC
    - keyboard
    - lid/accel/gyro/light/etc sensors
    - battery/charger
    - leds
    - etc
  - e.g., Nuvoton NPCX9M6F
    - <https://docs.zephyrproject.org/latest/boards/arm/npcx9m6f_evb/doc/index.html>
    - Cortex-M4F
    - 256 KB RAM and 64 KB boot ROM
    - external flash?  (some parts have on-chip 512KB/1MB flash?)
    - etc
  - GSC interconnects with EC as well
    - GSC can route the EC serial for CCD
    - GSC has WP and SPI pins to EC flash
    - etc
- firmware
  - it is zephyr-based
  - <https://chromium.googlesource.com/chromiumos/platform/ec/+/HEAD/docs/zephyr/README.md>
    - `zmake <board>`
    - `build/zephyr/<board>/output/zephyr.bin`
    - or, `emerge-<board> chromeos-base/chromeos-ec`
  - previously,
    - also <https://chromium.googlesource.com/chromiumos/platform/ec>
    - `make BOARD=<board>`
      - the binary will be at `build/<board>/ec.bin`
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

## Command Lines

- `ectool chargeoverride dontcharge` forces discharge
