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

## Kernel Image

- `src/scripts/build_kernel_image.sh` is used to create the kernel image
- the script generates these cmdline options
  - `console=`
  - `loglevel=${FLAGS_loglevel}`
  - `init=/sbin/init`
  - `cros_secure`
  - `drm.trace=0x106`
  - `root=${root_dev}`
  - `rootwait`
  - `ro`
  - `dm_verity.error_behavior=${FLAGS_verity_error_behavior}`
  - `dm_verity.max_bios=${FLAGS_verity_max_ios}`
  - `dm_verity.dev_wait=${dev_wait}`
  - `${device_mapper_args}`
    - `dm=\"1 vroot none ro 1,${table}\"`
  - `${FLAGS_boot_args}`
    - `noinitrd`
    - `cros_debug`
  - `vt.global_cursor_default=0`
  - `kern_guid=%U`
    - `%U` will be replaced by kernel partition uuid by depthcharge
  - `cros_lsb_release_hash=${FLAGS_version_attestation_hash}`
  - if x86,
    - `add_efi_memmap`
    - `noresume`
    - `i915.modeset=1`
  - board-specific `modify_kernel_command_line` can modify the cmdline
    - `tpm_tis.force=0`
    - `ramoops.ecc=1`
    - `intel_pmc_core.warn_on_s0ix_failures=1`
    - `xdomain=0`
    - `i915.enable_psr=1`
    - `swiotlb=65536`

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
  - `CONFIG_GOOGLE_SMI` provides `gsmi` platform driver
    - it is for coreboot `CONFIG_ELOG_GSMI`
    - elog is a flash-based event log on x86
    - the driver provides efivars to access the log
  - `CONFIG_GOOGLE_COREBOOT_TABLE` provides `coreboot_table` platform driver
    - it is the driver for coreboot table
    - the driver parses the coreboot table and registers devices listed in the
      table
    - `CONFIG_GOOGLE_MEMCONSOLE_COREBOOT` provides `memconsole` coreboot
      driver
      - the driver provides `/sys/firmware/log` to read the coreboot log
    - `CONFIG_GOOGLE_FRAMEBUFFER_COREBOOT` provides `framebuffer` coreboot
      driver
      - the driver registers a `simple-framebuffer` platform device
      - it is no longer used
    - `CONFIG_GOOGLE_VPD` provides `vpd` coreboot driver
      - the driver provides `/sys/firmware/vpd` to access VPD (vital product
        data)
- `CONFIG_TCG_TPM`
  - `CONFIG_TCG_TIS_SPI_CR50` provides `tpm_tis_spi` spi driver
    - it is for cr50-based tpm over spi
  - `CONFIG_TCG_CR50_I2C` provides `cr50_i2c` i2c driver
    - it is for cr50-based tpm over i2c
    - gsc (google security chip) is connected to the ap via i2c
- `CONFIG_CHROME_PLATFORMS`
  - `CONFIG_CHROMEOS_ACPI` provides `chromeos_acpi` platform driver
    - it is the driver for acpi `GGL0001` provided by coreboot
    - it exports acpi vals as attrs under
      `/sys/devices/platform/chromeos_acpi`
    - it replaces the downstream `CONFIG_ACPI_CHROMEOS` (`ChromeOS`) driver
  - `CONFIG_CHROMEOS_LAPTOP` matches the DMI table and registers i2c and acpi
    devices
    - it is no longer used
  - `CONFIG_CHROMEOS_PSTORE` provides `chromeos_pstore` platform driver
    - it matches ACPI/DMI and registers `ramoops` platform device
  - `CONFIG_CHROMEOS_TBMC` provides `chromeos_tbmc` platform driver
    - it is the driver for acpi `GOOG0006` provided by coreboot
    - it registers an input device to report `SW_TABLET_MODE`
  - `CONFIG_CROS_EC`
    - `CONFIG_CROS_EC_I2C` provides `cros-ec-i2c` i2c driver
      - it is no longer used
    - `CONFIG_CROS_EC_RPMSG` provides `cros-ec-rpmsg` rpmsg driver
      - it is used on mtk
    - `CONFIG_CROS_EC_ISHTP` provides `cros_ec_ishtp` ishtp driver
      - it is no longer used
    - `CONFIG_CROS_EC_SPI` provides `cros-ec-spi` spi driver
      - EC is commonly connected to ap via spi
    - `CONFIG_CROS_EC_UART` provides `cros-ec-uart` serdev driver
      - it is used on amd
    - `CONFIG_CROS_EC_LPC` provides `cros_ec_lpcs` platform driver
      - it is used on x86
    - they register `cros-ec-dev` platform device(s) to be driven by
      `CONFIG_MFD_CROS_EC_DEV`
  - `CONFIG_CROS_KBD_LED_BACKLIGHT` provides `chromeos-keyboard-leds` platform
    driver
  - `CONFIG_CROS_EC_CHARDEV` provides `cros-ec-chardev` platform driver
    - it exports `/dev/cros_ec`
  - `CONFIG_CROS_EC_LIGHTBAR` provides `cros-ec-lightbar` platform driver
    - it is no longer used
  - `CONFIG_CROS_EC_VBC` provides `cros-ec-vbc` platform driver
    - it is no longer used
  - `CONFIG_CROS_EC_DEBUGFS` provides `cros-ec-debugfs` platform driver
    - it exports `/sys/kernel/debug/cros_ec` for EC log
  - `CONFIG_CROS_EC_SENSORHUB` provides `cros-ec-sensorhub` platform driver
    - it registers sensor devices connected to the EC sensorhub, such as
      - `cros-ec-accel`
      - `cros-ec-baro`
      - `cros-ec-gyro`
      - `cros-ec-mag`
      - `cros-ec-prox`
      - `cros-ec-light`
      - `cros-ec-lid-angle`
  - `CONFIG_CROS_EC_SYSFS` provides `cros-ec-sysfs` platform driver
    - it exports EC info (`reboot`, `flashinfo`, `version`, etc.) to sysfs
  - `CONFIG_CROS_EC_TYPEC` provides `cros-ec-typec` platform driver
    - it manages typec ports via ec
  - `CONFIG_CROS_USBPD_LOGGER` provides `cros-usbpd-logger` platform driver
    - it logs PD events to kmsg
  - `CONFIG_CROS_USBPD_NOTIFY` provides `cros-usbpd-notify` or
    `cros-usbpd-notify-acpi` platform driver
    - it is the driver for acpi `GOOG0003`
  - `CONFIG_CROS_HPS_I2C`  provides `cros-hps` i2c driver
    - it is not used yet?
  - `CONFIG_CHROMEOS_PRIVACY_SCREEN` provides `chromeos_privacy_screen_driver`
    acpi driver
    - it is no longer used
  - `CONFIG_CROS_TYPEC_SWITCH` provides `cros-typec-switch` platform driver
    - it is not used yet?
  - `CONFIG_CROS_EC_PD_UPDATE` provides `cros-ec-pd-update` platform driver
    - it is downstream and is no longer used
- `CONFIG_MFD_CROS_EC_DEV` provides `cros-ec-dev` platform driver
  - `cros-ec-dev` platform devices are registered by `CONFIG_CROS_EC`
  - there can be several `cros-ec-dev` devices
    - `cros_ec`
    - `cros_fp` on x86
    - `cros_scp` on mtk
  - it adds a bunch of subdevices where individual drivers bind to
    - `cros-ec-sensorhub` if sensorhub
    - `cros-ec-rtc` if rtc (on qcom)
    - `cros-usbpd-charger`, `cros-usbpd-logger`, and `cros-usbpd-notify`
    - `cros-ec-pchg` if peripheral chargers
    - `cros-ec-chardev`, `cros-ec-debugfs`, and `cros-ec-sysfs`
- `CONFIG_KEYBOARD_CROS_EC` provides `cros-ec-keyb` platform driver
  - it supports a few special keys such as 
    - power, volume, brightness, screen lock
    - lid, tablet switch
- `CONFIG_I2C_CROS_EC_TUNNEL` provides `cros-ec-i2c-tunnel` platform driver
  - it is the driver for a pseudo i2c bus where i2c devices are connected to
    EC rather than AP
- `CONFIG_CHARGER_CROS_USBPD` provides `cros-usbpd-charger` platform driver
  - it is the driver for ec-based USB PD
- `CONFIG_CHARGER_CROS_PCHG` provides `cros-ec-pchg` platform driver
  - it is the driver for ec peripheral chargers
- `CONFIG_REGULATOR_CROS_EC` provides `cros-ec-regulator` platform driver
  - it is used on mtk
- `CONFIG_SND_SOC_CROS_EC_CODEC` provides `cros-ec-codec` platform driver
  - it is used on zork
- `CONFIG_RTC_DRV_CROS_EC` provides `cros-ec-rtc` platform driver
  - it is used on qcom
- `CONFIG_PWM_CROS_EC` provides `cros-ec-pwm` platform driver
  - it is used on arm
- `CONFIG_EXTCON_USBC_CROS_EC` provides `extcon-usbc-cros-ec` platform driver
  - it is used on kukui
- `CONFIG_IIO_CROS_EC_SENSORS_CORE`
  - `CONFIG_IIO_CROS_EC_SENSORS` provides `cros-ec-sensors` platform driver
    - it supports `cros-ec-accel`, `cros-ec-gyro`, and `cros-ec-mag` devices
  - `CONFIG_IIO_CROS_EC_SENSORS_LID_ANGLE` provides `cros-ec-lid-angle`
    platform driver
  - `CONFIG_IIO_CROS_EC_ACTIVITY` is downstream and is no longer used
  - `CONFIG_IIO_CROS_EC_SENSORS_SYNC` is downstream and is no longer used
  - `CONFIG_IIO_CROS_EC_ACCEL_LEGACY` is no longer used
  - `CONFIG_IIO_CROS_EC_LIGHT_PROX` provides `cros-ec-light-prox` platform
    driver
    - it supports `cros-ec-light` and `cros-ec-prox` devices
  - `CONFIG_IIO_CROS_EC_BARO` provides `cros-ec-baro` platform driver
    - it is no longer used
- `CONFIG_CROS_EC_MKBP_PROXIMITY` provides `cros-ec-mkbp-proximity` platform
  driver
  - it is no longer used
