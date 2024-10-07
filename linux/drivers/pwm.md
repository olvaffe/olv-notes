Kernel PWM
==========

## PWM Driver

- `devm_pwmchip_alloc` allocs a `pwm_chip`
- driver fills out `pwm_chip`
- `pwmchip_add` adds the `pwm_chip` to core
  - it is added to `pwm_chips`
  - `of_pwmchip_add` sets `of_xlate` to `of_pwm_xlate_with_flags`

## Consumer

- `devm_pwm_get` returns a `pwm_device`
  - `of_pwm_get` parses `pwms` and calls `fwnode_to_pwmchip` to find the chip
  - `of_pwm_xlate_with_flags` is called to get the `pwm_device`
- `pwm_init_state` inits `pwm_state`
- `pwm_apply_might_sleep` applies the state
