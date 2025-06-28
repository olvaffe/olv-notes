Flashrom
========

## Programmer

- Depending on the programmer, the transport mechanisms are different
  - for `linux_mtd`, it opens `/dev/mtd<N>`
  - for `pony_spi`, it opens `/dev/ttyS*`
  - for `dediprog`, it uses libusb to discover and open the programming
  - for `lspcon_i2c_spi`, it opens `/dev/i2c-<bus>` and does ioctls
  - for `it8212`, it uses libpciutils to discover the device, finds the BAR
    base, and mmaps through `/dev/mem`
    - others call `rget_io_perms` and do io ports
- some programmers support SPI; we need to drive them
- in many cases, the programmers are serial port like.  We need to drive
  them to do bit banging.

## CH341A

- EEPROM
  - 24xx I2C EEPROM
  - 25xx SPI EEPROM
    - e.g., Winbond W25Q128JV, 128Mb, 2.7V - 3.6V, 133MHz
- Pin 1 is chip selection
  - the flash has a dot mark for pin 1
  - the clip has a red wire for pin 1
  - the connector has a diagram for all pins
  - the programmer has a diagram for 24xx and 25xx, with a dot mark for pin 1
- `flashrom -p ch341a_spi -r backup` reads from flash
  - `Found Winbond flash chip "W25Q128.V" (16384 kB, SPI) on ch341a_spi.`
  - `Reading flash... done.`
  - if the clip is not properly installed, `No EEPROM/flash device found.`
    instead
