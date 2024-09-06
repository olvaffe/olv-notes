Kernel Regulator
================

## Regulators

- there are system-wide fixed-voltage regulators
  - these are typically driven by `regulator-fixed`
- there are also adjustable regulators for devfreq
  - regulators are commonly attached to SPMI or I2C bus of the soc
  - it is also common to have a PMIC attached to the SPI bus
    - a PMIC may be a MFD where some of the cells are regulators
- regulator drivers call `devm_regulator_register` to register regulators

## Consumers

- dt node, `&gpu`
  - `mali-supply = <&vdd_gpu_s0>;`
- `devm_regulator_get` calls `_regulator_get` to look up the regulator
  - `of_get_regulator` returns the phandle of the `%s-supply` prop (`mali` in
    this example)
  - `of_find_regulator_by_node` returns the regulator
