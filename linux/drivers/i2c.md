I2C
===

## Adapters and Clients

- an `i2c_adapter` represents an I2C bus controller
  - there is usually a `platform_driver` that allocates an adapter,
    initializes it, and calls `i2c_add_numbered_adapter` to add it to i2c core
  - each adapter has a `i2c_algorithm` that is responsible for programming the
    controller to send an `i2c_msg` over the bus
- an `i2c_client` repressents an I2C device
  - an `i2c_driver` binds to a client
  - the driver constructs `i2c_msg` and calls `i2c_transfer` to send the
    messages
- scanning happens when a new adapter is registerd in `i2c_register_adapter`
  - `of_i2c_register_devices`
  - `i2c_acpi_register_devices`
