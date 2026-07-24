# Kernel GPIO

## GPIO Controller

- `devm_gpiochip_add_data` registers a `gpio_chip`
  - it allocs a high-level `gpio_device` to wrap the low-level `gpio_chip`
  - `gpiochip_get_ngpios` inits `gc->ngpio` if the driver does not
  - `gdev->descs` is an array of `gpio_desc` for each pin
  - `gpiodev_add_to_list_unlocked` adds it to `gpio_devices`
  - `of_gpiochip_add`
    - `chip->of_xlate` defaults to `of_gpio_twocell_xlate` typically
  - `of_gpiochip_add_pin_range` parses `gpio-ranges` if pinctrl-based
  - `gpiochip_add_irqchip` adds irqchip if the gpio controller is also an irq
    controller
  - `gpiochip_setup_shared` allows two consumer drivers to share the same pin
  - `gpiochip_setup_dev` exposes the gpio chip to userspace
    - `gcdev_register` exposes the newer `/dev/gpiochipN` interface
    - `gpiochip_sysfs_register` exposes the legacy `/sys/class/gpio/gpiochipN`

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
      - drive mode is for output
        - push/pull: it can pull high/low itself
        - open drain: it can only pull low, and relies on external resistor to pull high
        - open source: it can only pull high, and relies on external resistor to pull low
      - bias is for input
        - pull-up: pull to high
        - pull-down: pull to low
        - disable: no pulling
    - `gpiod_line_state_notify` calls the notifiers
