Kernel GPIO
===========

## GPIO Controller

- `devm_gpiochip_add_data` registers a `gpio_device`
  - `gpiodev_add_to_list_unlocked` adds it to `gpio_devices`
  - `of_gpiochip_add`
    - `of_xlate` defaults to `of_gpio_simple_xlate`
  - `gpiochip_setup_dev`

## Consumers

- a consumer dt node should have
  - `led-gpios = <&gpio 15 GPIO_ACTIVE_HIGH>`
  - `power-gpios = <&gpio 1 GPIO_ACTIVE_LOW>;`
- `gpiod_get_index` returns a `gpio_desc`
  - `gpiod_find_by_fwnode` calls `of_find_gpio`
    - it parses `%s-gpios`
