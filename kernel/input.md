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
