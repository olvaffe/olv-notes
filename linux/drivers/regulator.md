Kernel Regulator
================

## Regulators

- <https://docs.kernel.org/power/regulator/overview.html>
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

## PMIC

- a pmic consists of multiple standard regulators
  - the goal of a regulator is to output stable voltage
  - types of regulators
    - pmos ldo, vin is slightly higher than vout
      - simple, no switching noise, but high heat
    - nmos ldo, vin can be much higher than vout?
    - buck, vin is higher than vout (step-down)
      - complex, switching noise, but low heat
    - boost, opposite of buck
- rk806 pmic
  - 10 bucks
    - vcc1..vcc10 are inputs
    - vout1..vcc10 are outputs
  - 6 pmos ldos
    - vcc11, vcc12, and vcca are inputs
    - pldo1..pldo5 and vccio are outputs
  - 5 nmos ldos
    - vcc13 and vcc14 are inputs
    - nldo1..nldo5 are outputs
  - orange pi 5 schematics
    - buck1: in `VCC_SYSIN`, out `VDD_GPU_S0`
    - buck2: in `VCC_SYSIN`, out `VDD_CPU_LIT_S0`
    - buck3: in `VCC_SYSIN`, out `VDD_LOGIC_S0`
    - buck4: in `VCC_SYSIN`, out `VDD_VDENC_S0`
    - buck5: in `VCC_SYSIN`, out `VDD_DDR_S0`
    - buck6: in `VCC_SYSIN`, out `VDD2_DDR_S3` and `VCC_1V1_NLDO_S3`
    - buck7: in `VCC_SYSIN`, out `VCC_2V0_PLDO_S3`
    - buck8: in `VCC_SYSIN`, out `VCC_3V3_S3`
    - buck9: in `VCC_SYSIN`, out `VDDQ_DDR_S0`
    - buck10: in `VCC_SYSIN`, out `VCC_1V8_S3` and `VCC1V8_PMU_DDR_S3`
    - pldo1: in `VCC_2V0_PLDO_S3`, out `VCC_1V8_S0`
    - pldo2: in `VCC_2V0_PLDO_S3`, out `VCCA_1V8_S0`
    - pldo3: in `VCC_2V0_PLDO_S3`, out `VDDA_1V2_S0`
    - pldo4: in `VCC_SYSIN`, out `VCCA_3V3_S0`
    - pldo5: in `VCC_SYSIN`, out `VCCIO_SD_S0`
    - vccio: in `VCC_SYSIN`, out `VCC_1V8_S3_PLDO6`
    - nldo1: in `VCC_1V1_NLDO_S3`, out `VDD_0V75_S3`
    - nldo2: in `VCC_1V1_NLDO_S3`, out `VDDA_DDR_PLL_S0`
    - nldo3: in `VCC_1V1_NLDO_S3`, out `VDDA_0V75_S0`
    - nldo4: in `VCC_1V1_NLDO_S3`, out `VDDA_0V85_S0`
    - nldo5: in `VCC_1V1_NLDO_S3`, no out
    - more broadly,
      - type-c power-in provides `VCC5V0_SYS`
      - `VCC5V0_SYS` is connected to
        - `VCC_SYSIN`, in parallel
        - `VCC_5V0`, in parallel
        - `VBUS_TYPEC`, via a switch which is on when a usb dev is connected
        - usb 2.0 type-a port
        - rtc
      - `VCC_SYSIN` is connected to
        - `VDD_CPU_BIG0_S0`, via a step-down regulator
        - `VDD_CPU_BIG1_S0`, via a step-down regulator
        - `VDD_NPU_S0`, via a step-down regulator
        - `VCC3V3_PCIE30`, via a step-down regulator when is on when a pci dev
          is connected
        - pmic
      - `VCC_5V0` is connected to
        - `VCC5V0_USB_HOST`, for usb 3.0 dev
        - `VCC5V_HDMI_TX0`, for hdmi
        - 26-pin header
        - green led
