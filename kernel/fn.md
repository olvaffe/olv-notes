ACPI and Fn
===========

Pressing a key could
1. trigger the keyboard, which gives a normal key
2. trigger the acpi
3. /proc/acpi/toshiba/keys

When acpi is triggered, anything could happen.  For example, an
ACPI_VIDEO_NOTIFY_INC_BRIGHTNESS event (see drivers/acpi/video.c) could happen.
And the driver could (optionally) increase the brightness, report the event
through /proc/acpi/event and/or input layer.

## kernel

* Some traditional acpi events are input events and go through input layer now
  * `KEY_POWER`, `KEY_SLEEP`, `KEY_SUSPEND`, `SW_LID`, and etc.
* netlink is prefered over `/proc/acpi/event`
