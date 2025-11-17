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

## `usbhid`

- `usbhid_probe` probes a usb hid device
  - `hid_allocate_device` allocs an hid dev
  - `hid_add_device` adds the hid dev

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

## 8BitDo SN30 Pro+

- when connected via USB,
  - Y+start: 057e:2009 Nintendo Co., Ltd Switch Pro Controller
  - X+start: 045e:028e Microsoft Corp. Xbox360 Controller
  - A+start: 054c:05c4 Sony Corp. DualShock 4 [CUH-ZCT1x]
  - B+start: 2dc8:6002 8BitDo 8BitDo SN30 Pro+
    - also just start or powered off
- as 8BitDo 8BitDo SN30 Pro+,
  - one `input_dev`
  - L / R: `BTN_TL / TR`
  - L2 / R2: `BTN_TL2 / TR2`
  - ABXY: `BTN_SOUTH / EAST / NORTH / SOUTH`
  - select / start: `BTN_SELECT / START`
  - home: `BTN_MODE`
  - star: no response
  - dpad: `ABS_HAT0X / HAT0Y`, with range [-1, 1]
  - left stick: `ABS_X / ABS_Y` with bad range?
  - right stick: `ABS_Z / ABS_RZ` with bad range?
  - `BTN_C / Z / THUMBL / THUMBR`???
- a good gamepad also has
  - force feedback (`EV_FF`)
  - gyroscope (`EV_ABS`)
  - accelerometer

## gamepad

- a cheap gamepad normally has
  - a d-pad (left, right, up, down)
  - a left circle pad (i.e., analog stick)
  - a right circle pad
  - four buttons on the right (A, B, X, Y)
  - another four buttons on the top side (L1, L2, R1, R2)
  - a select key
  - a start key
  - a switch to select between keyboard/mouse mode or joystick mode
- it can be driven by `hid-generic`
  - with three HID applications: keyboard, mouse, and joystick
  - three `input_dev`s are created
  - depending on the mode the gamepad is in, different keycodes are reported
    to different `input_dev`s
- when in keyboard/mouse mode,
  - d-pad reports `EV_KEY` and `KEY_LEFT`/`KEY_RIGHT`/`KEY_UP`/`KEY_DOWN`
  - left circle pad also reports `EV_KEY` and
    `KEY_LEFT`/`KEY_RIGHT`/`KEY_UP`/`KEY_DOWN`
  - right circle pad reports `EV_REL` and `REL_X`/`REL_Y`
    - the values for X/Y axes are in [-8, 8]
    - multiple events are reported for 2D directions (e.g., top-right)
  - A/B/X/Y reports `EV_KEY` and `KEY_A`/`KEY_B`/`KEY_X`/`KEY_Y`
  - L1/R1 reports `EV_KEY` and `BTN_LEFT`/`BTN_RIGHT`
  - L2/R2 reports `EV_KEY` and `KEY_L`/`KEY_R`
  - select reports `EV_KEY` and `KEY_K`
  - start reports `EV_KEY` and `KEY_S`
- when in joystick mode,
  - d-pad reports `EV_ABS` and `ABS_HAT0X`/`ABS_HAT0Y`
    - key down reports a value of -1 or 1 in X or Y direction
    - key up reports a value of 0
    - key down/up can report multiple events (e.g., both X 1 and Y -1 for
      bottom-right)
    - it is called hat switch traditionally (because switch resembles a
      Chinese hat)
  - A/B/X/Y reports `EV_KEY` and `BTN_SOUTH`/`BTN_EAST`/`BTN_NORTH`/`BTN_WEST`
  - L1/R1/L2/R2 reports `EV_KEY` and `BTN_TL`/`BTN_TR`/`BTN_TL2`/`BTN_TR2`
  - select reports `EV_KEY` and `BTN_SELECT`
  - start reports `EV_KEY` and `BTN_START`
  - left circle pad reports `EV_ABS` and `ABS_X`/`ABS_Y`
    - range is in [-128, 127]
  - right circle pad reports `EV_ABS` and `ABS_Z`/`ABS_RZ`
    - range is in [-128, 127]
