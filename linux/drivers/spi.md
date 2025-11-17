Kernel SPI
==========

## Controller Driver

- comparing to i2c, spi is faster but more expensive
- the controller driver calls `spi_register_controller` to register a
  controller
  - this automatically calls `of_register_spi_devices` to add all child nodes
    as `spi_device`s

## Device Driver

- `module_spi_driver` defines a device driver which calls
  `spi_register_driver`
