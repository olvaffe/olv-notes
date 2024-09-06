Kernel clk
==========

## Controllers

- an soc typically has several clock controllers, for different ip blocks
  - each clock controller has many basic clocks
  - an soc can have hundreds of basic clocks in total
- there are generic support for basic types of clocks
  - `clk_hw_register_fixed_rate` registers a fixed-rate clock
    - `fixed_rate` is the fixed rate in hz
  - `clk_hw_register_gate` registers a gate clock
    - `reg` is the iommu reg, `bit_idx` is the bit to gate/ungate
  - `clk_hw_register_mux` registers a multiplexer clock
    - it connects to several paraent clocks as inputs, and outputs one of them
    - `reg` is the iommu reg, `shift`/`mask` are the bits for the parent index
  - `clk_hw_register_fixed_factor` registers a fixed multiplier/divider clock
    - `mult` is the multiplier and `div` is the divider
  - `clk_hw_register_divider` registers an adjustable divider clock
    - `reg` is the iommu reg, `shift`/`width` are the bits for the divisor
  - `clk_hw_register_fractional_divider` registers a adjustable fractional
    divider clock
    - `reg` is the iommu reg, `mshift`/`mwidth` are the bits for the
      numerator, `nshift`/`nwidth` are the bits for the denominator
  - `clk_hw_register_composite` registers a composite clock
    - it seems to be a pseudo clock that combines a rate clock, a gate clock,
      and a mux clock
    - when the clock is enabled/disabled, it emulates using the gate clock
    - when the rate is changed, it emulates using the rate clock and the mux
      clock
- `clk_register` registers a clock
  - a `struct clk_core` is allocated and filled in from `hw->init`
    - `core->dev` points to the device, if any
    - `core->of_node` points to the OF node, if any
    - `core->hw` points to the hw
      - `hw->init` is cleared, since it is only for init
  - a `struct clk` is allocated
    - it represents a consumer of the clock
    - `hw->clk` points to the clock
  - `__clk_core_init` inits the clock
  - variants
    - `clk_hw_register` is similar, except it returns an error code rather
      than a `struct clk`
    - `of_clk_hw_register` is similar, except there is no `dev`
- a controller driver typically registers all its clocks and calls
  `of_clk_add_hw_provider`
  - this allocs a `of_clk_provider` and adds it to `of_clk_providers`
  - this allows a consumer to call `of_clk_get_by_name` later to look up a
    clock from the controller

## Consumers

- dt node, `gpu: gpu@fb000000`
  - `assigned-clocks = <&scmi_clk SCMI_CLK_GPU>;`
  - `assigned-clock-rates = <200000000>;`
  - `clocks = <&cru CLK_GPU>;`
  - `clock-names = "core";`
- `platform_probe` calls `of_clk_set_defaults` before probing
  - `assigned-clocks` and `assigned-clock-rates` are parsed
  - `of_clk_get_from_provider` looks up the clock from the provider
    - the controller driver should have register itself as a provider
  - `clk_set_rate` sets the initial clock rate
- the consumer driver calls `devm_clk_get` during probe
  - that calls `clk_get` which calls `of_clk_get_hw`
    - `of_parse_clkspec` parses `clocks` and `clock-names`
    - `of_clk_get_hw_from_clkspec` looks up the clock from the provider
  - `clk_hw_create_clk` allocs a `struct clk` for the `clk_hw`
