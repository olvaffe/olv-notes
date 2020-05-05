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
- 
