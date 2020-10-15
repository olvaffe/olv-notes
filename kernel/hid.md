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
