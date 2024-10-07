Kernel LEDs
===========

## Core

- `devm_led_classdev_register` registers a device
  - it parses OF for things such as `linux,default-trigger`
  - `led_trigger_set_default` sets the default trigger
- triggers call `led_trigger_register` to register themselves
  - it adds the trigger to `trigger_list`
