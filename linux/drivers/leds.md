Kernel LEDs
===========

## Core

- `devm_led_classdev_register` registers a device
  - it parses OF for things such as `linux,default-trigger`
  - `led_trigger_set_default` sets the default trigger
- triggers call `led_trigger_register` to register themselves
  - it adds the trigger to `trigger_list`

## `CONFIG_LEDS_GPIO`

- the dt node of `gpio-leds` looks like
  - `compatible = "gpio-leds";`
  - `pinctrl-names = "default";`
  - `pinctrl-0 = <&leds_gpio>;`
  - `led-1`
    - `gpios = <&gpio1 RK_PA2 GPIO_ACTIVE_HIGH>;`
    - `label = "status_led";`
    - `linux,default-trigger = "heartbeat";`
- `gpio_leds_create`
  - `priv->num_leds` is the number of child nodes
  - `devm_fwnode_gpiod_get` returns the `gpio_desc`
    - `of_find_gpio` parses `gpios` and finds `gpio_desc`
      - `RK_PA2` is the pin number (a gpio chip controls multiple pins)
      - `GPIO_ACTIVE_HIGH` is the flag
    - `gpiod_request` requests (exclusive ownership of) the pin
      - on soc, the gpio chip is typically backed by a pinctrl
      - `gpiochip_generic_request` calls `pinctrl_gpio_request`, which is
        often nop
    - `gpiod_configure_flags` configures the pin
      - because of `GPIOD_ASIS`, this does not change the direction
  - `led_init_default_state_get` parses `default-state`, or returns
    `LEDS_DEFSTATE_OFF`
  - `create_gpio_led` registers a `led_classdev`
    - `gpiod_direction_output` sets the gpio direction to output
      - on soc, this usually calls `pinctrl_gpio_direction_output`
    - `devm_led_classdev_register_ext` registers the led
      - this parses `linux,default-trigger` and inits the led
    - `devm_pinctrl_get_select_default` configures the pin function, if it is
      muxed
