SBC
===

## Popular SoCs

- <https://hackerboards.com/>
- AllWinner
  - A64, 2015, A53 x4
  - H5,  2016, A53 x4
  - H7,  2017, A53 x4
- Amlogic
  - S905X, 2015, A53 x4
  - S905X3, 2019, A55 x4
- Broadcom
  - BCM2711, 20xx, A72 x4
  - BCM2712, 20xx, A76 x4
- Intel
  - Celeron N4500, 2021, Tremont x2
  - N100, 2023, Gracemont x4
- MediaTek
  - MT7986 (Filogic 830), 20xx, A53 x4
  - MT7988A (Filogic 880), 20xx, A73 x4
- NXP
  - i.MX 8M, 2017, A53 x4
  - i.MX 8 QuadMax, 2017, A72 x2 + A53 x4
- Rockchip
  - RK3328, 2017, A53 x4
  - RK3399, 2016, A72 x2 + A53 x4
  - RK3566, 2021, A55 x4
  - RK3568, 2021, A55 x4
  - RK3588, 2022, A76 x4 + A55 x4
- <https://www.cpubenchmark.net/> single thread results
  - 14900K: 4791
  - N100: 1966
  - N4500: 1394
  - A76 @ 2.4GHz: 1047
  - A72 @ 2.4GHz: 830
  - A55 @ 1.8GHz: 286
  - A53 @ 1.8GHz: 236

## Popular SBCs

- FriendlyElec NanoPi
  - R2S: RK3328
  - R4S: RK3399
  - R5S: RK3568B2
  - R6S: RK3588S
- Raspberry Pi
  - 4: BCM2711
  - 5: BCM2712
- Raxda Rock
  - 5B: RK3588
- Sinovoip Banana Pi
  - BPI-R3: MT7986
  - BPI-R4: MT7988A
  - BPI-M7: RK3588
- Xunlong Orange Pi
  - 3: RK3566
  - 5: RK3588
- Hardkernel ODROID
 - M1S: RK3566

## NanoPI R5S

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
