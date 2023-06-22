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
  - `CONFIG_SERIAL_8250=y`
  - `CONFIG_SERIAL_8250_CONSOLE=y`
  - `CONFIG_SERIAL_8250_DW=y`
  - `CONFIG_VT`
  - `CONFIG_DRM_FBDEV_EMULATION`
  - `CONFIG_FRAMEBUFFER_CONSOLE`
- enable compressed modules and firmwares
  - `CONFIG_MODULE_COMPRESS=y`
  - `CONFIG_FW_LOADER_COMPRESS=y`
  - `CONFIG_EXTRA_FIRMWARE_DIR="/lib/firmware"`
- enable more filesystems
  - `CONFIG_BTRFS_FS`
  - `CONFIG_F2FS_FS`
  - `CONFIG_AUTOFS_FS` for systemd
    - <https://github.com/systemd/systemd/blob/main/README>
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
