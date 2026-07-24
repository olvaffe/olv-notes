# Kernel GPIO

## GPIO Controller

- `devm_gpiochip_add_data` registers a `gpio_device`
  - `gpiodev_add_to_list_unlocked` adds it to `gpio_devices`
  - `of_gpiochip_add`
    - `of_xlate` defaults to `of_gpio_simple_xlate`
  - `gpiochip_setup_dev`

## Consumers

- a consumer dt node should have, for example,
  - `led-gpios = <&gpio 15 GPIO_ACTIVE_HIGH>`
  - `power-gpios = <&gpio 1 GPIO_ACTIVE_LOW>;`
- consumer driver calls `devm_gpiod_get_optional` or a variant
  - `gpiod_get_index` calls `gpiod_find_and_request` to return a `gpio_desc`
    - `gpiod_fwnode_lookup` looks up the desc
      - `of_find_gpio`
        - it parses `%s-gpios`, `%s-gpio`, or `of_find_gpio_quirks`
        - `of_find_gpio_device_by_xlate` find the gpio dev
          - it loops through all gpio devices on global `gpio_devices` and
            matches using `of_gpiochip_match_node_and_xlate`
          - the controll driver has called `gpiochip_add_data` to update the
            list
            - `gpio_chip` is provided by the driver for hw access
            - `gpio_device` wraps a gpio chip to add kernel-tracked states
        - `of_xlate_and_get_gpiod_flags` returns the desc from the chip
      - `of_convert_gpio_flags` converts flags to `GPIO_x`
    - if no match, `gpiod_find` is the legacy method
      - it finds in `gpio_lookup_list`
      - the controller driver has called `gpiod_add_lookup_table` to update
        the list
    - `gpiod_request` requests the pin
      - it tests and sets `GPIOD_FLAG_REQUESTED`
      - `gpiod_hwgpio` returns the pin number
      - optional `gc->request` and `gc->get_direction` are called
    - `gpiod_configure_flags` configs the pin according to the flags
      - `lflags` is from dt
      - `dflags` is from the consumer driver
      - polarity
        - active high: high voltage means active
        - active low: low voltage means active
      - drive mode
        - push/pull: it can pull high/low itself
        - open drain: it can only pull low, and relies on external resistor to pull high
        - open source: it can only pull high, and relies on external resistor to pull low
      - bias
        - pull-up: if configured as input, pull to high
        - pull-down: if configured as input, pull to low
        - disable: if configured as input, no pulling
    - `gpiod_line_state_notify` calls the notifiers
