Chrome OS Kernel
================

## Kernel

- `cros_workon --board=${BOARD} start chromeos-kernel-5_15`
- make changes to `src/third_party/kernel/v5.15`
  - or fetch a WIP branch, `git fetch cros <branch>`
    - `https://chromium.googlesource.com/chromiumos/third_party/kernel/+log/<branch>`
- to build, `USE="kgdb vtconsole" emerge-${BOARD} --nodeps chromeos-kernel-5_15`
- to deploy, `./update_kernel.sh --remote=<DUT IP address> --ssh_port 22`

## Config

- to change kernel config
  - `chromeos/config/*/*.config`
- to update kernel cmdline
  - on DUT: `/usr/share/vboot/bin/make_dev_ssd.sh --edit_config`
  - `update_kernel.sh`: edit `src/build/images/<BOARD>/latest/config.txt`
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

## Build Kernel

- kernel config
  - follow <../../kernel/config.md>
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

## Pack

- fit image
  - <https://github.com/lentinj/u-boot/blob/master/doc/uImage.FIT/source_file_format.txt>
  - compress kernel
    - `lz4 DUT/arch/arm64/boot/Image Image.lz4`
  - dtb
    - `ln -sf DUT/arch/arm64/boot/dts/foo/bar.dtb Image.dtb`
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
- vboot image
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
- module tarball
  - `tar zcf Image.modules.tar.gz -C DUT/MODULES lib/modules`

## Deploy

- on dut
- deploy modules 
  - `sudo tar xkf Image.modules.tar.gz --no-same-owner -C /`
- deploy kernel
  - `sudo cp Image.vboot /dev/mmcblk1p4`
- boot the new kernel once
  - `sudo cgpt add -i 4 -S 0 -T 1 -P15 /dev/mmcblk1`
