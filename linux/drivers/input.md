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
