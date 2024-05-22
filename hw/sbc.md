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

## Flash

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
