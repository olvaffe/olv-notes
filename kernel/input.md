Input subsystem
===============

## Structs

* `struct input_event`: time/type/code/value
* `struct input_dev`: represent a input device.
    - include evbit, keybit, relbit, absbit, mscbit, ledbit, sndbit, ffbit and swbit.
    - scancode -> keycode map.
    - another set of long arrays for tracking states: key, led, snd, sw
    - first open calls ->open, last close calls ->close.  can also be notified about ->event (pcspkr)
    - input device could be grabbed, only the grabbing handle receives events
* `struct input_handler`: an interface of input device (evdev interface, blah)
* `struct input_handle`: a handle connects handler and device.
                       multiple devices might have the same interface.  There is one handler, but multiple handles.

## flow

* device driver calls `input_register_device`
* handler driver calls `input_register_handler`
* if a device/handler pair matches, `handler->connect` is called which should
  allocate and call `input_register_handle`
* when the handler is to be used (e.g., userspace opens /dev/input/eventX),
  `input_open_device` is called and device starts sending events.
* device report event through `input_report_XXX`, which calls `input_event`
* handler could also simulate events through `input_inject_event`.

## event types

* Types

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
* a touch on the touchpad might generate (`BTN_TOUCH, ABS_X, ABS_Y,
      ABS_PRESSURE, ABS_TOOL_WIDTH, BTN_TOOL_FINGER, BTN_LEFT, BTN_RIGHT, etc.`)
      and a `EV_SYN` follows.
* on the device side, (HW STATE -> series of EVENTS -> SYN)
      on the handler side, (series of EVENTS -> SYN -> compose HW STATE)

## device -> handler

* report events through one of `input_report_{key,rel,abs,blah} `
* they call into `input_event`, which calls `input_handle_event`
* `input_handle_event` might ignore the event, route the event to device and/or handler

## `EVIOCGRAB`

* invokes `evdev_grab`

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
