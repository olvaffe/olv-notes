Input subsystem
===============

## Driver

- an input device driver calls `input_allocate_device` and
  `input_register_device` to alloc and register the device
  - it should fill in `input_dev` for name, supported keys, event callback, etc.
- when the driver has an event to report,
  - `input_event(dev, EV_MSC, MSC_RAW, code)` reports the raw code
  - `input_event(dev, EV_MSC, MSC_SCAN, scancode)` reports the scan code
  - `input_event(dev, EV_KEY, keycode, value)` reports the key code
    - value is for press or release
  - `input_event(dev, EV_SYN, SYN_REPORT, 0)` marks the end of the report
  - helpers
    - `input_report_key` for `input_event(EV_KEY)`
    - `input_sync` for `input_event(EV_SYN, SYN_REPORT)`
- the input core can also send a request to the driver through the `event`
  callback
  - `EV_LED` to turn leds on/off
  - `EV_REP` to set repeat rate
- `input_event` calls `input_handle_event` in the core
  - `input_get_disposition` decides the disposition
  - `INPUT_IGNORE_EVENT` ignores the event
  - `INPUT_PASS_TO_DEVICE` calls `dev->event` callback
  - `INPUT_PASS_TO_HANDLERS` buffers the event in `dev->vals`
  - `INPUT_FLUSH` calls `handle->handler->events` to send `dev->vals` to all
    handlers

## atkbd and psmouse

- serio
  - `serio_init` registers `serio_bus`
  - when a `serio` port and a `serio_driver` match, `serio_connect_driver`
    connects the port to the driver
- i8042
  - `i8042_init` registers both `i8042` platform driver and device
    - `i8042_platform_init` also registers `i8042 kbd` and `i8042 aux` pnp
      drivers, but they are usually unused
    - `i8042_controller_check` performs pio to check the controller
  - `i8042_register_ports` calls `serio_register_port` to register the kbd
    and aux ports, if exists
- atkbd
  - `atkbd_init` calls `serio_register_driver` to register `atkbd_drv`
  - `atkbd_connect` connects to a `serio` port and calls
    `input_allocate_device`/`input_register_device` to alloc/register an
    `input_dev`
  - it speaks the ps2 protocol
    - `ps2_command` to send a cmd to the keyboard
    - `atkbd_receive_byte` to handle data from the keyboard and to send them
      to the core with `input_event`
    - the ps2 helper uses the `i8042` serio port to communicate with the hw
- psmouse
  - `psmouse_init` calls `serio_register_driver` to register `psmouse_drv`
  - the rest is similar to atkbd

## Handler

- a input event handler calls `input_register_handler` to register a handler
  - `id_table` and the `match` callback describe what devices the handler is
    interested in
  - `connect` is called for each matched device
    - the handler should call `input_register_handle` and `input_open_device`
      to connect to the device
  - on input events, `filter`/`event`/`events` handle the events
    - if `filter` returns true for an event, the event is removed from all
      following handlers
- handler could also simulate events through `input_inject_event`.
- example handlers
  - `rfkill_handler_init` toggles RF
    - it matches devices that have `KEY_WLAN`, `KEY_BLUETOOTH`,
      `KEY_RFKILL`, `SW_RFKILL_ALL`, etc.
  - vt `kbd_init` for vt input
    - it matches `EV_KEY`
  - `sysrq_register_handler` for sysrq
    - it matches `KEY_LEFTALT`
  - ledtrig `input_events_init` toggles leds on inputs
    - it matches `EV_KEY`, `EV_REL`, `EV_ABS`
  - `input_leds_init` provides controls for leds on input devices
    - it matches `EV_LED`
    - it does not handle events, but calls `led_classdev_register` to register
      a led device
    - when the led is toggled via the led device, it calls
      `input_inject_event(EV_LED)` to send the event to the device driver
  - `evdev_init` provides `/dev/input/event*`
    - it matches all input devices
    - it passes events to userspace clients
    - ioctl `EVIOCGRAB` calls `evdev_grab` and `input_grab_device`
      - all future events are passed only to the evdev handler and not to
        other handlers

## Structs

- `struct input_event`: time/type/code/value
- `struct input_dev`: represent a input device.
    - include evbit, keybit, relbit, absbit, mscbit, ledbit, sndbit, ffbit and swbit.
    - scancode -> keycode map.
    - another set of long arrays for tracking states: key, led, snd, sw
    - first open calls ->open, last close calls ->close.  can also be notified about ->event (pcspkr)
    - input device could be grabbed, only the grabbing handle receives events
- `struct input_handler`: an interface of input device (evdev interface, blah)
- `struct input_handle`: a handle connects handler and device.
                       multiple devices might have the same interface.  There is one handler, but multiple handles.

## event types

- Types

        EV_SYN		0x00
        EV_KEY		0x01
        EV_REL		0x02
        EV_ABS		0x03
        EV_MSC		0x04 /*misc */
        EV_SW		0x05
        EV_LED		0x11
        EV_SND		0x12
        EV_REP		0x14
        EV_FF		0x15
        EV_PWR		0x16
        EV_FF_STATUS	0x17
        EV_MAX		0x1f
        EV_CNT		(EV_MAX+1)
- a touch on the touchpad might generate (`BTN_TOUCH, ABS_X, ABS_Y,
      ABS_PRESSURE, ABS_TOOL_WIDTH, BTN_TOOL_FINGER, BTN_LEFT, BTN_RIGHT, etc.`)
      and a `EV_SYN` follows.
- on the device side, (HW STATE -> series of EVENTS -> SYN)
      on the handler side, (series of EVENTS -> SYN -> compose HW STATE)

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

## Userspace

- <https://gitlab.freedesktop.org/libevdev/libevdev>
  - libevdev is a low-level wrapper for evdev
- <https://gitlab.freedesktop.org/libevdev/evemu>
  - evdev provides tools to debug evdev
  - `evemu-describe` shows info about an evdev
  - `evemu-record` shows (optionally records) events for an evdev
  - `evemu-device` creates an uinput evdev
  - `evemu-play` generates a sequence of events
  - `evemu-event` generates an event
- <https://gitlab.freedesktop.org/libinput/libinput>
  - libinput is a high-level wrapper for evdev
  - it depends on
    - `libudev`
    - `mtdev`, for multi-touch
    - `libevdev`
    - `libwacom`
  - userspace compositors use libinput
- <https://gitlab.freedesktop.org/libinput/libei>
  - libei provides emulated input
  - userspace compositors may use libei, but the adoption rate is low
