Kernel Reset Controller
=======================

## Controller

- an soc often has a reset controller that controls the reset signals to
  various (hundreds) blocks
- the controller driver is expected to
  - fills in `reset_controller_dev`, where `ops` is `reset_control_ops`
    - `reset` asserts and deasserts
    - `assert` asserts the reset
    - `deassert` deasserts the reset
    - `status` returns the status
  - calls `devm_reset_controller_register` to register the controller
    - this adds the `reset_controller_dev` to `reset_controller_list`

## Consumer

- the consumer dt node often has something like
  - `resets = <&cru 3>;`, where `cru` is the controller node
  - `reset-names = "tx";`
- when the consumer driver calls `devm_reset_control_get`,
  `__of_reset_control_get` looks the controller up
  - it finds the idx from `reset-names`
  - it finds the controller node from `resets`
    - it falls back to `reset-gpios`
  - `__reset_find_rcdev` finds `reset_controller_list`
  - `rcdev->of_xlate` finds the reset line id
  - `__reset_control_get_internal` returns `reset_control` for the reset line
- when the consumer driver needs to reset the device,
  - `reset_control_assert` followed by `reset_control_deassert`, or
  - `reset_control_reset`
