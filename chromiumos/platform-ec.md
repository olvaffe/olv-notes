# Chrome OS EC Firmware

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

## Developer Mode

- EC does not know if developer mode is enable or not
  - the bit is stored in Ti50 TPM NVRAM

## Key Combos

- for clamshells,
  - Refresh+Power: hw reset EC
  - Refresh+Power+ESC: after hw reset, if EC detectes ESC, it signals AP to
    enter recovery mode
- for tablets or detachables,
  - power+vol-up+vol-down+10s: hw reset EC and signals AP to enter recovery mode
- power key
  - short press (200ms): power on, wake up from suspend, or notify OS
  - long press (10s): hard power off

## Kernel

- several mcus may run ec fw but provide different features
  - real ec provides many features
  - fpmcu provides `EC_FEATURE_FINGERPRINT`
  - intel ish provides `EC_FEATURE_ISH`
  - mtk scp provides `EC_FEATURE_SCP`
- kernel talks to ec via one of the tranports
  - `CONFIG_CROS_EC_SPI` matches `google,cros-ec-spi` on spi
  - `CONFIG_CROS_EC_LPC` matches `GOOG0004` on lpc
  - `CONFIG_CROS_EC_ISHTP` matches sensorhub uuid on intel ish (for intel ish)
  - `CONFIG_CROS_EC_RPMSG` matches `google,cros-ec-rpmsg` on rpmsg (for mtk scp)
- `CONFIG_MFD_CROS_EC_DEV` matches the platdev added by the transport drv and
  adds subdevices
  - if `EC_CMD_MOTION_SENSE_CMD`, add `cros-ec-sensorhub`
  - if `EC_FEATURE_GPIO`, add `cros-ec-gpio`
  - if `EC_FEATURE_RTC`, add `cros-ec-rtc`
  - if `EC_FEATURE_LED`, add `cros-ec-led`
  - if `EC_FEATURE_PWM_KEYB`, add `cros-keyboard-leds`
  - if `EC_FEATURE_CHARGER`, add `cros-charge-control`
  - if `EC_FEATURE_USB_PD` and `EC_FEATURE_UCSI_PPM`, add `cros_ec_ucsi`
    - new devs use `google,cros-ec-ucsi` or `GOOG0021` directly
  - if `EC_FEATURE_USB_PD` but not `EC_FEATURE_UCSI_PPM`, add
    `cros-usbpd-charger` and `cros-usbpd-logger`
  - if `EC_FEATURE_USB_PD`, add `cros-usbpd-notify`
  - if `EC_CMD_PCHG_COUNT`, add `cros-ec-pchg`
  - unconditionally add `cros-ec-chardev`, `cros-ec-debugfs`, etc.
- more subdevs described by dt/acpi directly
  - `CONFIG_I2C_CROS_EC_TUNNEL` matches `google,cros-ec-i2c-tunnel` and `GOOG0012`
  - `CONFIG_KEYBOARD_CROS_EC` matches `google,cros-ec-keyb` and `GOOG0007`
  - more
