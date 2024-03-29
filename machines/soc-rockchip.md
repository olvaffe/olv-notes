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
