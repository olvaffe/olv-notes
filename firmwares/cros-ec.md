Chrome OS EC
============

## Directory Structure

- <https://chromium.googlesource.com/chromiumos/platform/ec>
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

- kernel exposes `/dev/cros_ec` on DUT
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
- EC console is accessible from host
  - requires a debug table
  - read-only until CCD is opened

## Update EC firmware

- `make BOARD=<board>`
  - the binary will be at `build/<board>/ec.bin`
  - or, `emerge-<board> chromeos-ec`
- `flashrom` on DUT
  - `flashrom -p ec -r <backup.bin>`
  - `flashrom -p ec -w <path-to/ec.bin>`
- `flash_ec` script on host
  - requires working servod, see <cros-cr50.md>
  - `flash_ec --board=<boardname> [--image=<path/to/ec.bin>]`
- this flashes to EC flash, which is different from H1 flash
