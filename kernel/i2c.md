I2C
===

## Synaptics I2C Touchpad

- i2c bus
  - ACPI scan handler `lpss_handler` finds the `acpi_device` for the
    DesignWare I2C adapter and adds a corresponding `platform_device`
  - platform driver `dw_i2c_driver` binds to the platform device
  - `i2c_dw_probe` calls `i2c_add_numbered_adapter` to add the adapter
    - the adapter uses `i2c_dw_algo` as its `i2c_algorithm`
- i2c device
  - when an adapter is added, `of_i2c_register_devices` and/or
    `i2c_acpi_register_devices` are called to register i2c devices
  - `i2c_new_client_device` is called on each firmware node (acpi or of) to
    add `i2c_client`
  - in ACPI, any hid-over-i2c device has `PNP0C50` as its `_CID`
  - i2c driver `i2c_hid_driver` binds to the i2c client
  - `i2c_hid_probe` calls `hid_allocate_device` to allocate a `hid_device` and
    `hid_add_device` to add the device to the hid core
    - `hid_scan_report` scans the report descriptor and sets the device group
      to `HID_GROUP_MULTITOUCH`
- hid device
  - hid driver `mt_driver` binds to the `HID_GROUP_MULTITOUCH` hid device
  - `mt_probe` calls `hid_hw_start` with `HID_CONNECT_DEFAULT`
  - `hidinput_connect` calls `input_allocate_device` and
    `input_register_device` to add `input_dev`s

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
