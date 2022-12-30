Kernel MMC/SD/SDIO
==================

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
