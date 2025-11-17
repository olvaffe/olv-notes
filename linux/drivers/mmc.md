Kernel MMC/SD/SDIO
==================

## History

- sd specs
  - 1997, MultiMediaCards (MMCs)
  - 1999, SD 1.01
    - max size 2GB (standard capacity, SDSC)
    - max speed 12.5MB/s (default speed)
  - SD 1.10
    - max speed 25MB/s (high speed)
  - 2006, SD 2.0
    - max size 32GB (high capacity, SDHC)
  - 2010, SD 3.01
    - max size 2TB (extended capacity, SDXC)
    - max speed 50MB/s at 100MHz and 104MB/s at 208MHz (UHS-I)
      - doubled when DDR
  - 2013, SD 4.1
    - max speed 156MB/s or 312MB/s (UHS-II)
  - 2016, SD 5.0 and 5.1
  - 2017, SD 6.0
    - max speed 312MB/s or 624MB/s (UHS-III)
  - 2018, SD 7.0
    - max size 128TB (ultra capacity, SDUC)
    - max speed 985MB/s (SD Express)
  - 2020, SD 8.0
    - max speed 1969MB/s or 3938MB/s
  - 2022, SD 9.0
  - 2023, SD 9.1
- card speed class ratings
  - C classes (original)
    - C2, min write speed 2MB/s
    - C4, min write speed 4MB/s
    - C6, min write speed 6MB/s
    - C10, min write speed 10MB/s
  - U classes (UHS)
    - U1, min write speed 10MB/s
    - U3, min write speed 30MB/s
  - V classes (Video)
    - V6, min write speed 6MB/s
    - V10, min write speed 10MB/s
    - V30, min write speed 30MB/s
    - V60, min write speed 60MB/s
    - V90, min write speed 90MB/s
  - E classes (Express)
    - E150, min write speed 150MB/s
    - E300, min write speed 300MB/s
    - E450, min write speed 450MB/s
    - E600, min write speed 600MB/s
- performance ratings
  - A classes
    - A1, read 1500 ipos and write 500 ips
    - A2, read 4000 ipos and write 2000 ips
- inside an sd card, there are
  - an sd controller,
  - one or more nand chips
- eMMC
  - a MMC card in a BGA IC package, including
    - a mmc controller and a nand chip
  - UFS slowly replaces eMMC since 2016
  - 2011, eMMC 4.5
    - read 140MB/s write 50MB/s
  - 2013, eMMC 5.0
    - read 250MB/s write 90MB/s
  - 2015, eMMC 5.1
    - read 250MB/s write 125MB/s
  - 2019, eMMC 5.1A

## SDIO spec

- Formats
  - MMC, SD, SDIO
  - CF
  - MemoryStick
- SD deprecates MMC
  - MMC has seven-pin interface
  - SD and SDIO have nine-pin interface
  - SD is compatible with MMC?
- SD supports 1-bit, 4-bits, and SPI transfer modes
  - All SD cards must support all three modes, except microSD where SPI is
    optional.
  - MMC supports 1-bit and SPI.
  - The cards should support clock freq. up to 25MHz for regular cards, and
    50MHz for high-speed cards.
- An SDIO card is a combination of an SD card and an I/O device
  - GPS receiver, WiFi or BT adapter, camera, ...
- An SDIO card is compatible with SDHC in that inserting an SDIO card into a
  non SDIO-aware host will not cause any damange.
- SPI is mandatory to SDIO, unlike to SD.
  - SPI has 4 pins.
  - No in-band addressing (relies on out-of-band chip select)
  - Interrupt is out-of-band

## Broadcom

- Polled (throughput/cpu): 13.07/64.74
  HW Enab: 15.25/63.13
  TX Lazy+ RX GLOM: 19.34/37.49
  Zero Copy DMA: 20.62/41.37
- use 512 bytes block size
- glomming to send multiple packets in one SDIO transaction

## Directory Structures

- Under `drivers/mmc`.
- `core/` is the core.
  - The core registers two buses: mmc and sdio
- `host/` is the host drivers.
- `card/` is the card drivers.
  - SDIO cards/devices have classes just like usb devices.

## Card Insersion

- When there is an interrupt, the host driver calls `mmc_detect_change`.
  - It schedules `mmc_rescan`.
- In the case there is no `bus_ops` yet, `mmc_rescan` will
  call `mmc_attach_sdio`, `mmc_attach_sd`, or `mmc_attach_mmc`, which attaches
  the `bus_ops` and do any initialization.
  - It simply calls `bus_ops->detect` if there is already `bus_ops`.

## Important types

- Host controller is represented by `struct mmc_host`.
  - A host has a single `struct mmc_card *card`.  It is set directly by one of
    the attaching functions.
- In `mmc_attach_sdio`,
  - A card is allocated and initialized.
  - The functions of the card are allocated and initialized.
  - Then the card is `mmc_add_card` to the device model.
  - And each function is `sdio_add_func` to the device model too.
- SDIO function is represented by `struct sdio_func`.
  - It has `struct mmc_card *card`, which is the card it belongs to.
  - It has a `struct device *`.
  - It has `class`, `vendor`, and `device` ids.
    - `class` is from FBR, Function Basic Registers.
    - The rest is from CIS of code CISTPL_MANFID.
  - It has `struct sdio_func_tuple *tuples`, which stores CISs (Card Information
    Structure).
    - The tuple is `code`, `size` of the data, and `data`.
