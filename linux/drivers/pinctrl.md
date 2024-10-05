Linux pinctrl
=============

## Controller

- a pin controller has
  - multiple gpio banks, such as 5
  - each gpio bank has several io muxes, such as 4
  - each io mux has several pins, such as 8
  - there is a total of `5*4*8=160` pins
- the dt node usually has
  - child nodes for the hw gpio banks
    - these child nodes have `compatible` and are bound by a gpio driver
  - child nodes to group the pins
    - e.g., we want to use two pins for uart
      - the uart is called a "function" and is controlled by `pinmux_ops` 
      - we could use pin 0,1 or pin 2,3, etc. for the uart function
      - each of the possibilities is called a "group" and is controlled by
        `pinctrl_ops`
    - this way, a pin can be used for several functions
      - which pins are used for which functions are described by dt
- `pinctrl_desc` describes a pin controller
  - `pinctrl_ops *pctlops` controls pin grouping
  - `pinmux_ops *pmxops` controls pin muxing
  - `inconf_ops *confops` controls pin configuration
  - `pinctrl_pin_desc *pins` is an array of pin descs
- `devm_pinctrl_register` registers a `pinctrl_dev` from `pinctrl_desc`

## RK3588S

- looking at the datasheet,
  - the soc has more than 1000 pins
  - some pins have fixed functions
    - e.g., pin A1 is `VSS_1`
  - some pins are muxed and can be configured for different functions
    - e.g., pin AA41 can be `EMMC_D2`, `FSPI_D2_M0`, or `GPIO2_D2_u`
  - among the muxed pins, some can be configured as gpio
- according to `rk3588_pin_ctrl`,
  - there are 5 gpio banks
  - each gpio bank has 32 pins
- `rockchip_pinctrl_parse_groups` parses `rockchip,pins`
  - the 4 numbers are bank, pin, mux (func), and config
  - `<2 RK_PD2 1 ...>` configures pin AA41 for `EMMC_D2`
  - `<2 RK_PD2 2 ...>` configures pin AA41 for `FSPI_D2_M0`
