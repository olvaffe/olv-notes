Chrome OS EC
============

## EC Firmware, Zephyr EC

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
- in my experience, the latest EC firmware does not boot
  - one should use a known-good version
  - a version is already packed into `chromeos-firmwareupdate`
    - `chromeos-firmwareupdate --unpack` on DUT to unpack
- some EC has built-in flash and some has external flash

## Flash EC firmware

- `flashrom` on DUT
  - `flashrom -p ec -r <backup.bin>`
  - `flashrom -p ec -w <path-to/ec.bin>`
  - or, `chromeos-firmwareupdate`, but it never works for me
- `flash_ec` script on host
  - requires working servod, see <cros-cr50.md>
  - `flash_ec --board=<boardname> [--zephyr] [--image=<path/to/ec.bin>]`
  - it might need `npcx_monitor.bin` as well
- this flashes to EC flash, which is different from H1 flash

## `ectool`

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
