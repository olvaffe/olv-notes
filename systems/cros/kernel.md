Chrome OS Kernel
================

## Kernel

- `cros_workon --board=${BOARD} start chromeos-kernel-5_15`
- make changes to `src/third_party/kernel/v5.15`
  - or fetch a WIP branch, `git fetch cros <branch>`
    - `https://chromium.googlesource.com/chromiumos/third_party/kernel/+log/<branch>`
- to build, `USE="kgdb vtconsole" emerge-${BOARD} --nodeps chromeos-kernel-5_15`
- to deploy, `./update_kernel.sh --remote=<DUT IP address> --ssh_port 22`
  - this pushes the latest image under `/build/$BOARD/boot` to dut

## Config

- <https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/refs/heads/main/eclass/cros-kernel>
  - these are mainly for use with upstream kernels
- <https://chromium.googlesource.com/chromiumos/third_party/kernel/+/refs/heads/chromeos-5.15/chromeos/>
  - these are what cros uses
  - which branch is board-dependent
  - `CHROMEOS_KERNEL_FAMILY=chromeos ./chromeos/scripts/prepareconfig chromiumos-qualcomm`
    cats these 3 files
    - `chromeos/config/chromeos/base.config`
    - `chromeos/config/chromeos/arm64/common.config`
    - `chromeos/config/chromeos/arm64/chromiumos-qualcomm.flavour.config`
- ebuild
  - ebuild uses `cros-kernel2`, which is at
    `src/third_party/chromiumos-overlay/eclass/cros-kernel2.eclass`
  - it picks `$CHROMEOS_KERNEL_SPLITCONFIG` or
    `chromeos-$CHROMEOS_KERNEL_ARCH` as the config
    - on chipset-mt8183, we have
      `CHROMEOS_KERNEL_SPLITCONFIG="chromiumos-mediatek"` and
      `CHROMEOS_KERNEL_ARCH="arm64"`
  - `chromeos/scripts/prepareconfig` generates the final config using
    `chromeos/configs`
    - `base.config` is common between all cros kernels
    - `$arch/common.config` is common between those of the same arch
    - `$arch/chromiumos-$soc.flavour.config` is common between those of the
      same soc
  - the config is also modified based on USE flags
    - see `CONFIG_FRAGMENTS` in the eclass file
    - some interesting ones are
      - `USE=pcserial` and `USE=lpss_uart` for x86 serial console
      - `USE=fbconsole` `USE=vtconsole` for fb/vt console
- to change kernel config
  - `chromeos/config/*/*.config`
- to update kernel cmdline
  - on DUT: `/usr/share/vboot/bin/make_dev_ssd.sh --edit_config`
  - `update_kernel.sh`: edit `src/build/images/<BOARD>/latest/config.txt`

## `CONFIG_LOW_MEM_NOTIFY`

- `get_available_mem_adj` returns the "available" memory in pages
  - free pages (usually very low)
  - clean file cache pages (usually very high and very quick to reclaim)
  - anon pages divided by a weight (usually very high but need swapping out)
- `low_mem_thresholds` is an array of thresholds in ascending order
  - the kernel notifies the userspace whenever the avail mem decreases and
    crosses thresholds
  - the system is considered low-mem when the the avail mem is less than the
    first/lowest threshold
- `/sys/kernel/mm/chromeos-low_mem`
  - `available` shows of `get_available_mem_adj` in MBs
    - with a light load, this reports 2.6GB on a 4GB machine
  - `margin` shows `low_mem_thresholds` in MBs
    - it reports `201 1548` on a 4GB machine
  - `ram_vs_swap_weight` reports 4
- `low_mem_check` is called from `__alloc_pages` to check the current avail
  mem against thresholds
  - when the system enters low-mem, it prints `entering low_mem` and the
    available ram
  - userspace is notified, in the hope that userspace frees up memory before
    the kernel oom-killer kicks in
- oom score
  - each task is assigned a score of badness
    - 0 is never kill
    - 1000 is always kill
  - the score can be adjusted by `oom_score_adj`
  - oom kills the one with high scores
- sysrq-x
  - kills the chrome session to trigger a crashdump
  - prints `sysrq: Cros dump and crash`

## Linux Distro: Config

- to boot cros kernel with regular distro, there are a few caveats
- enable serial console, vt, and fbcon
  - `CONFIG_SERIAL_8250`
  - `CONFIG_SERIAL_8250_CONSOLE`
  - `CONFIG_SERIAL_8250_DW`
  - `CONFIG_VT`
  - `CONFIG_DRM_FBDEV_EMULATION`
  - `CONFIG_FRAMEBUFFER_CONSOLE`
- enable compressed modules and firmwares
  - `CONFIG_MODULE_COMPRESS`
  - `CONFIG_FW_LOADER_COMPRESS`
  - `CONFIG_FW_LOADER_COMPRESS_ZSTD`
  - `CONFIG_EXTRA_FIRMWARE_DIR="/lib/firmware"`
- enable more filesystems
  - `CONFIG_BTRFS_FS`
  - `CONFIG_F2FS_FS`
  - `CONFIG_VFAT_FS` (built-in this and NLS modules)
  - `CONFIG_AUTOFS4_FS` for systemd
    - <https://github.com/systemd/systemd/blob/main/README>
- misc
  - `CONFIG_ZRAM` (built-in this and lzo)
- disable security for simplicity
  - `CONFIG_SECURITY`
  - `CONFIG_SECURITY_CHROMIUMOS_READONLY_PROC_SELF_MEM`
- disable werror
  - `CONFIG_WERROR`

## Linux Distro: Build

- kernel config
  - follow <../../linux/config.md>
    - `make alldefconfig ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- O=DUT`
- build
  - `make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- O=DUT`
    - these targets are included
      - `vmlinux`
      - `modules`
      - `dtbs`
      - `Image.gz`
- install modules
  - `make modules_install INSTALL_MOD_PATH=MODULES INSTALL_MOD_STRIP=1 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- O=DUT`

## Linux Distro: Pack

- fit image (arm64)
  - <https://github.com/lentinj/u-boot/blob/master/doc/uImage.FIT/source_file_format.txt>
  - create symlinks
    - `ln -sf arch/arm64/boot/Image`
    - `ln -sf arch/arm64/boot/dts/foo/bar.dtb Image.dtb`
  - compress kernel
    - `lz4 Image Image.lz4`
  - dtb
  - `dtc -I dts -O dtb -p 1024 -o Image.fit Image.its`, where `Image.its` is

    /dts-v1/;
    / {
        images {
                kernel@1 {
                        data = /incbin/("Image.lz4");
                        type = "kernel_noload";
                        arch = "arm64";
                        os = "linux";
                        compression = "lz4";
                        load = <0>;
                        entry = <0>;
                };
                fdt@1 {
                        data = /incbin/("Image.dtb");
                        type = "flat_dt";
                        arch = "arm64";
                        compression = "none";
                        hash@1 {
                                algo = "sha1";
                        };
                };
        };
        configurations {
                conf@1 {
                        kernel = "kernel@1";
                        fdt = "fdt@1";
                };
        };
    };
- vboot image (arm64)
  - dummy bootloader
    - `dd if=/dev/zero of=Image.bl bs=512 count=1`
  - edit `Image.cmdline`
    - `root=PARTUUID=%U/PARTNROFF=1`, to use the next partition after the
      kernel partition.  depthcharge replaces `%U` by the UUID of the kernel
      partition
    - `rootwait root=/dev/mmcblk1p3`
    - `console=ttyMSM0,115200 console=tty1`
  - `futility vbutil_kernel --pack Image.vboot \
       --vmlinuz Image.fit \
       --config Image.cmdline \
       --bootloader Image.bl \
       --version 1 \
       --arch aarch64 \
       --keyblock /usr/share/vboot/devkeys/kernel.keyblock \
       --signprivate /usr/share/vboot/devkeys/kernel_data_key.vbprivk`
- vboot image (x86)
  - kernel is `arch/x86/boot/bzImage`
  - bootloader must be `bootstub.efi`
    - <https://chromium.googlesource.com/chromiumos/third_party/bootstub>
    - `/lib64/bootstub/bootstub.efi` in chroot
  - arch is `x86-64`
- module tarball
  - `tar zcf Image.modules.tar.gz -C DUT/MODULES lib/modules`

## Linux Distro: Deploy

- on dut
- deploy modules 
  - `sudo tar xkf Image.modules.tar.gz --no-same-owner -C /`
  - `depmod -a <version>`
- deploy kernel
  - `sudo cp Image.vboot /dev/mmcblk1p4`
- boot the new kernel once
  - `sudo cgpt add -i 4 -S 0 -T 1 -P15 /dev/mmcblk1`

## Drivers

- `CONFIG_GOOGLE_FIRMWARE`
  - `CONFIG_GOOGLE_SMI` is the driver for coreboot `CONFIG_ELOG_GSMI`
    - elog is a flash-based event log on x86
    - the driver provides efivars to access the log
  - `CONFIG_GOOGLE_COREBOOT_TABLE` is the driver for coreboot table
    - the driver parses the coreboot table and registers devices listed in the
      table
    - `CONFIG_GOOGLE_MEMCONSOLE_COREBOOT` is the driver for `memconsole`
      - the driver provides `/sys/firmware/log` to read the coreboot log
    - `CONFIG_GOOGLE_FRAMEBUFFER_COREBOOT` is the driver for `framebuffer`
      - the driver registers a `simple-framebuffer` platform device
      - coreboot does not seem to advertise the device anymore
    - `CONFIG_GOOGLE_VPD` is the driver for `vpd`
      - the driver provides `/sys/firmware/vpd` to access VPD (vital product
        data)
- `CONFIG_TCG_TPM`
  - `CONFIG_TCG_TIS_SPI_CR50` is the driver for cr50-based tpm over spi
    - it does not appear to be needed anymore
  - `CONFIG_TCG_CR50_I2C` is the driver for cr50-based tpm over i2c
    - gsc (google security chip) is connected to the ap via i2c
- `CONFIG_CHROME_PLATFORMS`
  - `CONFIG_CHROMEOS_ACPI` is the driver for acpi `GGL0001` provided by
    coreboot
    - it exports acpi vals as attrs under
      `/sys/devices/platform/chromeos_acpi`
    - it replaces the downstream `CONFIG_ACPI_CHROMEOS` driver
  - `CONFIG_CHROMEOS_LAPTOP` matches the DMI table and registers i2c and acpi
    devices
    - it does not appear to be needed anymore
  - `CONFIG_CHROMEOS_PSTORE` matches ACPI/DMI and registers `ramoops` platform
    device
  - `CONFIG_CHROMEOS_TBMC` is the driver for acpi `GOOG0006` provided by
    coreboot
    - it registers an input device to report `SW_TABLET_MODE`
    - it does not appear to be needed anymore
  - `CONFIG_CROS_EC` registers the `cros-ec-dev` platform device
    - `CONFIG_CROS_EC_I2C`
      - it does not appear to be needed anymore
    - `CONFIG_CROS_EC_RPMSG`
      - EC is connected to ap via rpmsg on arm (in addition to spi)
    - `CONFIG_CROS_EC_SPI`
      - EC is connected to ap via spi
    - `CONFIG_CROS_EC_CHARDEV` provides `/dev/cros_ec`
    - `CONFIG_CROS_EC_LIGHTBAR` does not appear to be needed anymore
    - `CONFIG_CROS_EC_VBC` does not appear to be needed anymore
    - `CONFIG_CROS_EC_DEBUGFS` exports `/sys/kernel/debug/cros_ec` for EC log
    - `CONFIG_CROS_EC_SENSORHUB` registers sensor devices connected to the
      EC sensorhub
    - `CONFIG_CROS_EC_SYSFS` exports EC info (`reboot`, `flashinfo`,
      `version`, etc.) to sysfs
    - `CONFIG_CROS_EC_PD_UPDATE` provides autoupdate of PD firmware
    - `CONFIG_CROS_USBPD_LOGGER` is the driver for logging PD events to kmsg
    - `CONFIG_CROS_USBPD_NOTIFY` is the driver for acpi `GOOG0003`
    - `CONFIG_CROS_TYPEC_SWITCH`  is the driver for acpi `GOOG001A`
  - `CONFIG_CROS_HPS_I2C`  is the driver for acpi `GOOG0020` provided by coreboot
  - `CONFIG_CHROMEOS_PRIVACY_SCREEN` is the driver for acpi `GOOG0010`
- `CONFIG_MFD_CROS_EC_DEV` is the driver for `cros-ec-dev`
  - it adds a bunch of subdevices where individual drivers bind to
- `CONFIG_KEYBOARD_CROS_EC` provides keyboard support
  - EC is evolved from keyboard controller
- `CONFIG_I2C_CROS_EC_TUNNEL` is the driver for a pseudo i2c bus where i2c
  devices are connected to EC rather than AP
- `CONFIG_CHARGER_CROS_USBPD` is the driver for ec USB PD
- `CONFIG_CHARGER_CROS_PCHG` is the driver for ec peripheral chargers
- `CONFIG_REGULATOR_CROS_EC` is the driver for ec regulator
- `CONFIG_SND_SOC_CROS_EC_CODEC` is the driver for ec codec
- `CONFIG_RTC_DRV_CROS_EC` is the driver for ec rtc
- `CONFIG_PWM_CROS_EC` is the driver for ec pwm
- `CONFIG_EXTCON_USBC_CROS_EC` is the driver for ec usbc
- `CONFIG_IIO_CROS_EC_SENSORS_CORE`
  - `CONFIG_IIO_CROS_EC_ACCEL_LEGACY`
  - `CONFIG_IIO_CROS_EC_SENSORS_CORE`
  - `CONFIG_IIO_CROS_EC_SENSORS`
  - `CONFIG_IIO_CROS_EC_SENSORS_LID_ANGLE`
  - `CONFIG_IIO_CROS_EC_ACTIVITY`
  - `CONFIG_IIO_CROS_EC_SENSORS_SYNC`
  - `CONFIG_IIO_CROS_EC_LIGHT_PROX`
  - `CONFIG_IIO_CROS_EC_BARO`
- `CONFIG_CROS_EC_MKBP_PROXIMITY` is the driver for ec proximity sensor
