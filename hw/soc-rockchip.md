Rockchip SoCs
=============

## Products

- RK18 series
  - RK1808, A35 x2 @1.6GHz
- RK30 series
  - RK3036, A7 x2 @1.0GHz
- RK31 series
  - RK3126, A7 x4 @1.2GHz
  - RK3128, A7 x4 @1.2GHz
  - RK3188, A9 x4 @1.6GHz
- RK32 series
  - RK3288, A17 x4 @1.8GHz
  - RK3289, A7 x4 @1.5GHz
- RK33 series
  - RK3308, A35 x4 @1.3GHz
  - RK3326, A35 x4 @1.5GHz
  - RK3328, A53 x4
  - RK3368, A53 x8 @1.5GHz
  - RK3399, A72 x2 @1.8GHz + A53 x4 @1.4GHz
  - RK3399Pro, A72 x2 @1.8GHz + A53 x4 @1.4GHz
- RK35 series
  - RK3566, A55 x4
  - RK3568, A55 x4 @2.0GHz
  - RK3568J, A55 x4
  - RK3582, A76 x2 + A55 x4, no GPU
  - RK3588, A76 x4 + A55 x4
- RK Power series
  - PX2, A9 x2 @1.4GHz
  - PX3, A9 x4 @1.6GHz
  - PX30, A35 x4
  - PX5, A53 x8 @1.5GHz

## NanoPI R5S Boot Sequence

- <https://wiki.pine64.org/wiki/RK3399_boot_sequence>
  - BootRom (BROM)
    - after reset, cpu0 executes bootrom stored in the read-only memory
    - bootrom tries to load the bootloader from, in order,
      - nor or nand flash on spi1
        - it issues 0x9f (Read Identification)
        - it uses 0x03 (nor) or 0x13 (nand) to read blocks
        - it looks for for the bootloader magic number in the first 32 bytes
      - emmc
        - it looks for the bootloader magic number at sector 0x40
      - sd on sdmmc
        - same as emmc after sdmmc controller initialization
      - over usb by putting the OTG0 usb controller in device mode (id
        `2207:330c`)
        - the board usually has a maskrom button, which forces all prior
          methods to fail and forces bootrom to enter this mode
    - the bootloader consists of up to 3 parts
      - id block, which is the header
      - first stage, which is loaded to and executed from sram
        - it's main job is to initialize dram
      - optional second stage, which is loaded to and executed from low dram
        - it's main job is to chainload another bootloader
        - the first stage can choose to do both jobs without returning to
          bootram
      - if over usb, the host sends the first stage using request 0x0471 and the
        second stage using request 0x0472
  - U-Boot as the bootloader
    - it consists of 4 parts
      - TPL, which is loaded by bootrom to sram
        - it initializes dram, and returns to bootrom
      - SPL, which is loaded by bootrom to low dram
        - it loads respective parts of TF-A (ARM Trusted Firmware-A) BL31
          firmware to dram, sram, and pmu sram
        - it also loads u-boot proper to dram
      - TF-A BL31
        - it sets up EL2 to run u-boot proper
        - it stays resident until system shutdown
      - U-Boot proper
- <https://opensource.rock-chips.com/wiki_Rockusb>
  - bootrom has a minimal rockusb implementation
    - it only supports `rkdeveloptool db` (`DownloadBoot`), which tells
      bootrom to run the specified bootloader
  - the proprietary bootloader (`rkxx_loader_vx.xx.bin`) has two rockusb
    implementations
    - it consits of `ddr.bin`, `usbplug.bin`, and `miniloader.bin`
    - after `rkdeveloptool db rkxx_loader_vx.xx.bin`, the device will run
      `ddr.bin` to initialize dram and run `usbplug.bin` to enter rockusb mode
    - `miniloader.bin` has another rockusb implementation that is entered when
      - the recovery key or the volume up key is pressed, or
      - it cannot find the next bootloader at sector 0x4000
  - u-boot provides anothe rockusb implementation
    - `rockusb 0 mmc 0` to enter
- <https://opensource.rock-chips.com/wiki_Boot_option>
  - make sure bootrom is in rockusb/maskrom mode
  - `rkdeveloptool db rkxx_loader_vx.xx.bin` to initialize dram (`ddr.bin`)
    and enter another rockusb mode (`usbplug.bin`)
  - `rkdeveloptool wl 0x40 idbloader.img` to write `idbloader.img` to sector
    0x40
    - it consists of the idblock header, `u-boot-tpl.bin`, and
      `spl/u-boot-spl.bin`
    - or, if proprietary, `rkdeveloptool ul rkxx_loader_vx.xx.bin`
      - it writes `ddr.bin` and `miniloader.bin` to sector 0x40
  - `rkdeveloptool wl 0x4000 u-boot.itb` to write `u-boot.itb` to sector
    0x4000
    - it consists of `u-boot-nodtb.bin` (u-boot proper), `bl31.elf` (tf-a
      bl31), and `u-boot.dtb` (device tree)
  - `rkdeveloptool wl 0x8000 boot.img` to write `boot.img` to sector 0x8000
    - it consists of another bootloader (grub2) or the kernel
      image/dtb/initramfs
  - `rkdeveloptool wl 0x40000 rootfs.img` to write `rootfs.img` to sector
    0x40000
    - it is the root fs

## NanoPI R5C Debian

- <https://github.com/inindev/nanopi-r5>
  - `dtb/make_dtb.sh`
    - it downloads `linux-6.5.6.tar.xz` and extracts
      - `include/dt-bindings`
      - `include/uapi`
      - `arch/arm64/boot/dts/rockchip`
    - it applies `patches`
    - it uses `dtc` to generate `rk3568-nanopi-{r5c,r5s}.dtb`
  - `uboot/make_uboot.sh`
    - it clones <https://github.com/u-boot/u-boot.git> and checks out
      `v2023.10`
    - `cherry_pick` cherry-picks a bunch of upstream fixes
    - it applies `patches`
    - it builds u-boot twice, for r5c and r5s
      - `make nanopi-${model}-rk3568_defconfig`
      - `make BL31="$atf_file" ROCKCHIP_TPL="$tpl_file"`
        - the firmwares are from <https://github.com/rockchip-linux/rkbin>
        - this builds `idbloader.img` and `u-boot.itb`
  - `debian/nanopi-r5c/make_debian_img.sh`
    - it downloads
      - <https://mirrors.edge.kernel.org/pub/linux/kernel/firmware/linux-firmware-20230210.tar.xz>
      - <https://github.com/inindev/nanopi-r5/releases/download/v12.0.1/idbloader-r5c.img>
      - <https://github.com/inindev/nanopi-r5/releases/download/v12.0.1/u-boot-r5c.itb>
      - <https://github.com/inindev/nanopi-r5/releases/download/v12.0.1/rk3568-nanopi-r5c.dtb>
    - `parition_media` creates a single partition starting at sector 0x8000
    - `format_media` formats the partition to ext4
    - `mount_media` moutns the partition
    - it populates the partition with
      - `/etc/kernel-img.conf`
      - `/etc/fstab`
      - `/etc/kernel/postinst.d/dtb_cp`
        - it symlinks `/boot/rk3568-nanopi-r5c.dtb-${version}` to
          `/boot/rk3568-nanopi-r5c.dtb`
      - `/etc/kernel/postrm.d/dtb_rm`
        - it removes `/boot/rk3568-nanopi-r5c.dtb-${version}`
      - `/boot/mk_extlinux`
        - it updates `/boot/extlinux/extlinux.conf` when kernel is updated
          - `fdt /boot/rk3568-nanopi-r5c.dtb-${kver}`
          - `append root=... ro rootwait`
      - `/etc/kernel/postinst.d/update_extlinux`
      - `/etc/kernel/postrm.d/update_extlinux`
    - it adds to the partition these downloaded files
      - `/usr/lib/firmware/{rockchip,rtl_bt,rtl_nic,rtlwifi,rtw88,rtw89}`
      - `/boot/rk3568-nanopi-r5c.dtb`
    - it debootstraps with these packages
      - `linux-image-arm64, dbus, dhcpcd, libpam-systemd, openssh-server, systemd-timesyncd`
      - `rfkill, wireless-regdb, wpasupplicant`
      - `curl, pciutils, sudo, unzip, wget, xxd, xz-utils, zip, zstd`
    - it modifies/adds these files
      - `/etc/apt/sources.list`
      - `/etc/default/locale`
      - `/etc/wpa_supplicant/wpa_supplicant.conf`
      - `/usr/lib/dhcpcd/dhcpcd-hooks/10-wpa_supplicant`
      - `/etc/skel/.bashrc`
      - `/root/.bashrc`
      - `/etc/hostname`
      - `/etc/hosts`
    - it adds user `debian` and adds `/etc/sudoers.d/debian`
    - it adds `/etc/rc.local` for first-boot init and changes some files
      - the script generates `/etc/machine-id`, reconfigures ssh, resizes
        rootfs partition, changes partition uuid, generates macs, etc.
      - the script removes itself when done
    - it dds `idbloader-r5c.img` to sector 0x40 and dds `u-boot-r5c.itb` to
      0x4000
      - sector size is 512 bytes

## Orange Pi 5

- <https://github.com/orangepi-xunlong/orangepi-build>
  - it appears to be based on armbian
  - `scripts/main.sh`
    - `BOARD` is `orangepi5`
    - sources `external/config/boards/orangepi5.conf`
      - `BOARDFAMILY` is `rockchip-rk3588`
      - `BOOT_SCENARIO` is `spl-blobs`
      - `BOOT_SUPPORT_SPI` is `yes`
    - sources `scripts/configuration.sh`
      - sources `external/config/sources/families/rockchip-rk3588.conf`
        - sources `external/config/sources/families/include/rockchip64_common.inc`
          - `DDR_BLOB` is `rk35/rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.16.bin`
            - from <https://github.com/armbian/rkbin>
          - `BL31_BLOB` is `rk35/rk3588_bl31_v1.45_20240422.elf`
            - from <https://github.com/orangepi-xunlong/rk-rootfs-build/tree/rkbin/rk35>
            - it is prebuilt of <https://github.com/ARM-software/arm-trusted-firmware>
        - `KERNELSOURCE` is <https://github.com/orangepi-xunlong/linux-orangepi.git>
        - `KERNELBRANCH` is `orange-pi-6.1-rk35xx`
        - `LINUXCONFIG` is `linux-rockchip-rk3588-current`
        - `BOOTSOURCE` is <https://github.com/orangepi-xunlong/u-boot-orangepi.git>
        - `BOOTBRANCH` is `v2017.09-rk3588`
        - `BOOTCONFIG` is `orangepi_5_defconfig`
        - `IMAGE_PARTITION_TABLE` is `gpt`
        - `OFFSET` is `30`
        - `BOOTFS_TYPE` is `fat`
        - `ROOTFS_TYPE` is `ext4`
    - `compile_uboot` builds uboot
    - `compile_kernel` builds kernel
- manual
  - <https://github.com/rockchip-linux/rkbin>
  - <https://github.com/ARM-software/arm-trusted-firmware.git>
    - `CROSS_COMPILE=aarch64-linux-gnu- make PLAT=rk3588 bl31`
    - `build/rk3588/release/bl31/bl31.elf` is the image
  - <https://github.com/u-boot/u-boot.git>
    - `make orangepi-5-rk3588s_defconfig`
    - `ROCKCHIP_TPL=rkbin/bin/rk35/rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.16.bin
       BL31=arm-trusted-firmware/build/rk3588/release/bl31/bl31.elf
       CROSS_COMPILE=aarch64-linux-gnu- make`
    - `u-boot-rockchip.bin` and `u-boot-rockchip-spi.bin` are the images
      - `arch/arm/dts/rockchip-u-boot.dtsi`
      - `mkimage` packs `rockchip-tpl` (the ddr blob from rkbin) and
        `u-boot-spl` (`spl/u-boot-spl.bin`) into `idbloader.img`
      - `u-boot.itb` is u-boot proper
  - flash to sdcard
    - `dd if=u-boot-rockchip.bin of=/dev/sda seek=64`
  - flash to spi flash
    - `dd if=u-boot-rockchip.bin of=/dev/mtdblock0`
- frankstein image
  - image layout
    - LBA 0..33: GPT
    - LBA 64..16383: `idbloader.img`
    - LBA 16384..32767: `u-boot.itb`
    - LBA 32768..: free
  - `fallocate -l 256M a.img`
  - `echo -e 'label:gpt\nfirst-lba:34\nstart=64,size=16320\nstart=16384,size=16384\nstart=32768' | sfdisk a.img`
    - the third partition should be ESP for uboot to consider it bootable
  - `dd if=src.img of=a.img bs=512 skip=64 seek=64 count=16320 conv=notrunc`
  - `dd if=src.img of=a.img bs=512 skip=16384 seek=16384 count=16384 conv=notrunc`
  - `losetup -fP && mkfs.vfat /dev/loop0p3 && losetup -D`
    - populate `/extlinux/extlinux.conf`, kernel, rootfs, etc.
- serial
  - `minicom -D /dev/ttyUSB0 -b 1500000`
- u-boot log from latest rkbin/u-boot/atf
  - `rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.16.bin` packed in `idbloader.img`
    - from `DDR 9fffbe1e78 cym 24/02/04-10:09:20,fwver: v1.16`
    - to `change to F0: 2112MHz`
  - `u-boot-spl.bin` packed in `idbloader.img`
    - from `U-Boot SPL 2024.10-rc6 (Oct 01 2024 - 17:05:11 -0700)`
    - to `## Checking hash(es) for Image atf-2 ... sha256+ OK`
  - `bl31.elf` packed in `u-boot.itb`
    - from `NOTICE:  BL31: v2.11.0(release):v2.11.0-702-gbccc22756`
    - to `NOTICE:  BL31: Built : 16:46:29, Oct  1 2024`
  - u-boot packed in `u-boot.itb`
    - from `U-Boot 2024.10-rc6 (Oct 01 2024 - 17:05:11 -0700)`
    - to `Starting kernel ...`
    - `Hit any key to stop autoboot:  0`
      - autoboot executes `bootcmd`, which is `bootflow scan -lb`
- u-boot `printenv`
  - `include/env_default.h`
    - `baudrate=1500000`, from `CONFIG_BAUDRATE`
    - `bootcmd=bootflow scan -lb`, from `CONFIG_BOOTCOMMAND`
    - `bootdelay=2`, from `CONFIG_BOOTDELAY`
    - `loadaddr=0xc00800`, from `CONFIG_SYS_LOAD_ADDR`
  - `include/configs/rk3588_common.h`
    - `boot_targets=mmc1 mmc0 nvme scsi usb pxe dhcp spi`, from `BOOT_TARGETS`
    - `fdt_addr_r=0x12000000`
    - `fdtfile=rockchip/rk3588s-orangepi-5.dtb`, from
      `CONFIG_DEFAULT_FDT_FILE`
    - `fdtoverlay_addr_r=0x12100000`
    - `kernel_addr_r=0x02000000`
    - `kernel_comp_addr_r=0x0a000000`
    - `kernel_comp_size=0x8000000`
    - `partitions=...`, from `PARTS_DEFAULT`
      - `uuid_disk=${uuid_gpt_disk};`
      - `name=loader1,start=32K,size=4000K,uuid=${uuid_gpt_loader1};`
      - `name=loader2,start=8MB,size=4MB,uuid=${uuid_gpt_loader2};`
      - `name=trust,size=4M,uuid=${uuid_gpt_atf};`
      - `name=boot,size=112M,bootable,uuid=${uuid_gpt_boot};`
      - `name=rootfs,size=-,uuid=B921B045-1DF0-41C3-AF44-4C6F280D3FAE;`
    - `pxefile_addr_r=0x00e00000`
    - `ramdisk_addr_r=0x12180000`
    - `script_offset_f=0xffe000`
    - `script_size_f=0x2000`
    - `scriptaddr=0x00c00000`
- kernel log
  - `Booting Linux on physical CPU 0x0000000000 [0x412fd050]`
  - `Linux version 6.1.43-rockchip-rk3588 ...`
  - `Kernel command line: console=ttyS1,1500000 console=tty0 debug root=/dev/mmcblk1p3 rootwait...`
- maskrom
  - press the maskrom key and power on the board
    - the bootrom boots into the maskrom instead of the normal flow
  - <https://github.com/rockchip-linux/rkdeveloptool.git>
    - `autoreconf -i && ./configure && make` to build `rkdeveloptool`
  - <https://github.com/rockchip-linux/rkbin>
    - `./tools/boot_merger RKBOOT/RK3588MINIALL.ini` to generate
      `rk3588_spl_loader_v1.16.113.bin`
  - `./rkdeveloptool db rk3588_spl_loader_v1.16.113.bin` tells the bootrom to
    boot the specified miniloader
  - `./rkdeveloptool ef` tells the miniloader to erase the spi flash
    - the normal flow will find the stage1 from emmc rather than from spi
- uart in downstream kernel
  - `rk3588s.dtsi` defines `uart[0-9]`
    - they differ in `reg`, `interrupts`, `clocks`, `dmas`, and `pinctrl-0`
  - `rk3588s-orangepi-5.dtsi`
    - it enables `uart9` and changes `pinctrl-0` from `<&uart9m1_xfer>` to
      `<&uart9m2_xfer &uart9m2_ctsn>`
  - `rk3588s-orangepi-5.dts`
    - it keeps `uart{0,1,3,4}` disabled and changes `pinctrl-0` from
      - `<&uart0m1_xfer>` to `<&uart0m2_xfer>`
      - `<&uart1m1_xfer>` to `<&uart1m1_xfer>`
      - `<&uart3m1_xfer>` to `<&uart3m0_xfer>`
      - `<&uart4m1_xfer>` to `<&uart4m0_xfer>`
  - `rk3588-linux.dtsi`
    - it adds an `fiq-debugger` node with `<&uart2m0_xfer>`
  - `overlay/rk3588-uart2-m0.dts`
    - it disables `fiq-debugger` and enables `uart2` with `<&uart2m0_xfer>`
    - `/dev/ttyFIQ0` becomes `/dev/ttyS2` after applying
      `fdtoverlays /dtb/rockchip/overlay/rk3588-uart2-m0.dtbo`

## RK3588S Devicetree

- GMAC
  - mac implements layer 2 and phy implements layer 1, interconnected by mii
    - gmac is an impl of mac
    - mdio is an impl of mii
  - rk3588s has `gmac1` while rk3588 has both `gmac0` and `gmac1`
  - `gmac1: ethernet@fe1c0000` has a child `mdio1: mdio`
  - `rk3588s-orangepi-5.dts` adds a child node to `mdio1`, which is a phy and
    is compatible with `ethernet-phy-ieee802.3-c22`
- CRU, clock and reset unit
  - the datasheet says
    - Support total 18 PLLs to generate all clocks
    - One oscillator with 24MHz clock input
  - `cru: clock-controller@fd7c0000`
  - `rk3588_clk_init` inits the clock controller
    - there are 3 `rockchip_cpuclk_reg_data` (3 cpu clusters?)
      - `rk3588_cpub0clk_data` for 2 big cores
      - `rk3588_cpub1clk_data` for the other 2 big cores
      - `rk3588_cpulclk_data` for the 4 little cores
    - `rk3588_clk_branches` is a big table
      - fixed, top, bigcore0, bigcore1, dsu, audio, bus, center isp1, npu,
      - nvm, php, rga, vdec, sdio, usb, vdpu, venc, vi, vo0, vo1, pmu, more?
      - gpu has
        - `CLK_GPU_SRC`, downstream of `gpll_cpll_aupll_npll_spll_p`
        - `CLK_GPU_PVTM` and `CLK_CORE_GPU_PVTM`, downstrem of `CLK_GPU_SRC`
        - `CLK_GPU`, `CLK_GPU_COREGROUP`, and `CLK_GPU_STACKS`
          - downstream of `CLK_GPU_SRC`
          - panthor enables all 3 clocks
- PMU, power management unit
  - the datasheet says
    - Support 10 separate voltage domains
    - Support 45 separate power domains
  - `pmu: power-management@fd8d8000`
    - `power: power-controller`
      - the 45 pm domins form a tree
  - `include/dt-bindings/power/rk3588-power.h` lists 44 pm domins
    - `VD_LITDSU` has `RK3588_PD_CPU_{0,1,2,3}`
    - `VD_BIGCORE0` has `RK3588_PD_CPU_{4,5}`
    - `VD_BIGCORE1` has `RK3588_PD_CPU_{6,7}`
    - `VD_NPU` has `RK3588_PD_{NPU,NPUTOP,NPU1,NPU2}`
    - `VD_GPU` has `RK3588_PD_GPU`
    - `VD_VCODEC` has `RK3588_PD_{VCODEC,RKVDEC0,RKVDEC1,VENC0,VENC1}`
    - `VD_DD01` has `RK3588_PD_DDR01`
    - `VD_DD23` has `RK3588_PD_DDR23`
    - `VD_LOGIC` has `RK3588_PD_{CENTER,VDPU,RGA30,AV1,VOP,VO0,VO1,VI,ISP1}`
      - and `RK3588_PD_{FEC,RGA31,USB,PHP,GMAC,PCIE,NVM,NVM0,SDIO,AUDIO}`
      - and `RK3588_PD_{SECURE,SDMMC,CRYPTO,BUS}`
    - `VD_PMU` has `RK3588_PD_PMU1`
  - `rockchip_pm_domain_probe` inits with `rk3588_pm_domains`
    - it only lists 29 domains
    - i guess the rest is controlled by other means or left at default
