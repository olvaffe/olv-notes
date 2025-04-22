Kernel Kconfig
==============

## Overview

- `make *config` executes this goal

    %config: outputmakefile scripts_basic FORCE
            $(Q)$(MAKE) $(build)=scripts/kconfig $@
- `scripts/kconfig/Makefile`

## Tips

- boot issues
  - initramfs `/init` script requires `CONFIG_BINFMT_SCRIPT=y`
  - `systemd-udevd` requires `CONFIG_DEVTMPFS` and `CONFIG_UNIX`
  - vt console requires a drm driver and `CONFIG_DRM_FBDEV_EMULATION`
    - simple drm requires `CONFIG_SYSFB_SIMPLEFB` and `CONFIG_DRM_SIMPLEDRM`
  - emergency shell requires  `CONFIG_KEYBOARD_ATKBD=y` to interact
  - rootfs requires a block driver and a filesytem (and NLS)
- No `/dev/dri` on rpi
  - `echo 0x1ff > /sys/module/drm/parameters/debug`
  - `[drm:vc5_hdmi_init_resources [vc4]] ERROR Failed to get HDMI state machine clock`
    - make sure `vc4` is loaded after `raspberrypi-clk`
  - `[drm:vc4_hdmi_bind [vc4]] Failed to get ddc i2c adapter by node`
    - make sure `vc4` is loaded after `i2c-brcmstb`
- finding drivers
  - `./scripts/dtc/dt_to_config <dts>`
  - on a working kernel, for each device under `/sys/devices`, look for
    - `driver` is the driver of the device
    - `subsystem` is the bus the device is on
    - `modalias` is the magic string modprobe uses to load a module
      - on platform bus, it is formed from
        - of compatible string,
        - acpi `_HID` and `_CID`, or
        - hardcoded name
    - `of_node` is the corresponding of node
      - `dtc -I fs /sys/firmware/devicetree/base` to dump the entire of tree
    - `firmware_node` is the corresponding acpi node
  - `find /sys/devices -name driver | xargs readlink -e | sort | uniq`
    - this lists all drivers that are bound to some devices
    - no unused driver
    - modules that discover and create the devices but are not themselves
      drivers are not included
  - it is easy to find module names from here
    - some drivers does not have `module` links because they are built-in and
      they fail to set `device_driver::mod_name`
