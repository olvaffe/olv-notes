HID
===

## `hid-generic`

- `hid-generic` is a `hid_driver` that can drive any `hid_device` on the HID
  bus
  - e.g., a `hid_device` is created when a bluetooth device with HIDP (HID
    profile) is connected
- it calls `hid_hw_start` from hid core
  - the device might report multiple applications (keyboard, mouse, joystick,
    etc.)
  - `hidinput_connect` allocates and registers a `input_dev` for each
    application
    - for a gamepad, there is usually a hw switch to select between
      keyboard/mouse/joystick mode for the gamepad.  The mode decides which
      button maps to which keycode

## `i2c_hid_acpi`

- `i2c_hid_acpi_driver` is an `i2c_driver`
  - it matches `ACPI0C50` or `PNP0C50`
- `i2c_hid_acpi_probe`
  - `i2c_hid_acpi_get_descriptor` calls into acpi to get the descriptor addr
  - `i2c_hid_core_probe` from the i2c-hid core probes the device at the addr
    - `hid_allocate_device` allocates a new `hid_device`
    - `hid_add_device` adds the hid device to the hid bus, `hid_bus_type`
    - the core talks `HID Over I2C Protocol` with the deivce
- hid drivers match hid devices on the hid bus
  - e.g., `hid-multitouch` matches hid devices of group id
    `HID_GROUP_MULTITOUCH`
    - the hid device can be connected to via usb, bt, or i2c in this case
