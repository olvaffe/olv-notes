Kernel MTD
==========

## Flash Memory

- <https://en.wikipedia.org/wiki/Computer_memory>
  - volatile memory
    - DRAM, for primary memory
    - SRAM, for cpu cache
  - non-volatile memory (NVM)
    - Mask ROM, read-only
    - PROM, can be programmed once with special programmers
      - aka OTP-ROM, eFuse, etc.
    - EPROM, similar to PROM but can be erased for reprogramming with UV
    - EEPROM, can be erased and reprogrammed without special programmers
    - Flash, a modern type of EEPROM
- NOR vs NAND Flash Memory
  - cost per bit: 3 vs 1
  - random read speed: 250 vs 1
  - write speed: slow vs fast
  - erase speed: 1 vs 150
  - bit flipping: less vs more
  - bad block: less vs more
  - data retention: 20yr vs 10yr
  - application: code execution vs data storage
    - nor can be executed-in-place
- SLC vs MLC vs TLC NAND Flash
  - bits per cell: 1 vs 2 vs 3
  - p/e cycle: 100k vs 3k vs 1k
  - data retention: 10yr vs 1yr vs 1yr
  - read speed: 4 vs 2 vs 1
  - write speed: 20 vs 2 vs 1
  - erase speed: 4 vs 2 vs 1
- commonly found memory in computers
  - NOR Flash
    - small, reliable, XIP (execute in place)
    - ideal for bios/uefi/firmware
    - I guess there was a time when bios is stored in EEPROM and data is
      stored in RTC/CMOS (volatile and requires a battery)?
  - NAND Flash
    - raw nand for storage
  - eMMC
    - nand-behind-ftl-controller for storage
- <https://en.wikipedia.org/wiki/Boot_ROM>
  - on x86, the boot rom is typcailly stored in nor
    - it can be flashed and can lead to brick
    - it consists of early init (e.g., coreboot, proprietary, etc.) and
      payload (e.g., edk2, depthcharge, etc.)
    - it chainloads bootloader (e.g., systemd-boot, grub, etc.)
  - on arm, the boot rom is typically stored in mask rom
    - it cannot be flashed and cannot lead to brick
    - it consists of early init and payload
    - payload exhausts boot options and tries to find the bootloader in
      - eeprom or nor, typically behind spi controller
      - emmc, behind emmc controller
      - sd card, behind sd/mmc controller
      - usb storage, behind usb controller

## Subsystems

- mmc, for nand flash chip behind an sd controller
- ufs, for nand flash chip behind a ufs controller
- nvme, for nand flash chip behind an nvme controller
- usb-storage, for nand flash chip behind a usb controller
- nvmem, for eeprom, efuse, otp, etc.
- mtd, for nand or nor flash chips without a controller
  - it usually needs a sw middle layer for wear-leveling (FTL), block device
    emulation, etc.
- file systems
  - jffs2, works on mtd directly without middle layer
  - ubifs, works on mtd directly with ubi middle layer
  - f2fs, works on nand behind a controller

## Old

NAND:
- page: 256 or 512 bytes (or bigger), readable and programmable
- block: 4KB, 8KB or 16KB (or bigger), erasable and programmable
- address and data on the same bus; slow random read; uncapable of random write
- e.g., a 512Mbit nand flash consists of 128 * 1024 512b pages
- the real size of a page is 512+16, where the 16bytes is OOB

Scenerio: we have a 2M NOR mapped in CPU memory address space.

- describe the address/size of the NOR to maps/physmap.c
- physmap creates a map_info and pass it to, say, chips/map_rom.c, which gives mtd_info
- int add_mtd_device(struct mtd_info *mtd) is called to register the mtd_info to the core
- core notifies all notifiers, like mtd_blkdevs and mtdchar.

The controller and storage might be different.  But it can be the same:

- devices/mtdram.c
