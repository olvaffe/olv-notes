Kernel OPP
==========

## DT

- dt defines global opp tables, where each node has
  - `compatible = "operating-points-v2";`
  - optional `opp-shared;` means the opp table is shared by multiple devices
  - a list of opp nodes, where each node has
    - `opp-hz`
    - `opp-microvolt`
    - `clock-latency-ns`
    - `opp-suspend`
- opp core does not parse the global opp tables
  - instead, a device node can have `operating-points-v2` prop to refer to an
    opp table
  - when the device driver probes, it parses the opp table on demand
- when a device driver probes,
  - `devm_pm_opp_*` calls `_add_opp_table` to alloc the opp table
    - if this is the first device using the opp table,
      - `_allocate_opp_table` allocs the opp table
      - the new opp table is added to the global `opp_tables`
    - if this is the second device using a shared opp table,
      - `_managed_opp` returns the shared opp table
      - `_add_opp_dev` adds the device to `opp_table->dev_list`
    - if this is called a second time for the same device,
      - `_find_opp_table_unlocked` returns the table from `opp_tables`
    - `_update_opp_table_clk` inits `opp_table->clk` if any
  - `dev_pm_opp_set_*` calls `dev_pm_opp_set_config` to config the opp table
    - `_opp_set_clknames` inits `opp_table->clks` if any
      - `clk_get` gets clks
    - `_opp_set_regulators` inits `opp_table->regulators` if any
      - `regulator_get_optional` gets regulators
  - `dev_pm_opp_of_add_table` adds the opp table
    - `_of_add_opp_table_v2` parses dt opp table node
      - `_opp_allocate` allocates an opp
      - dt opp node is parsed
- when the device driver calls `dev_pm_opp_set_rate` to change opp,
  - `_find_freq_ceil` returns the target opp for the target freq
  - `_set_opp_level` calls `dev_pm_domain_set_performance_state` if the device
    is attached to a pmdomain
  - `_set_opp_bw` calls `icc_set_bw` if the device has an interconnect
  - if `opp_table->config_regulators`, the default
    `_opp_config_regulator_single` calls `regulator_set_voltage_triplet`
  - if `opp_table->config_clks`, the default `_opp_config_clk_single` calls
    `clk_set_rate`

## Dynamic OPP Table

- a device driver can build the opp table dynamically
  - as oppose to parsing the static opp table defined in dt
- also, when a device node has a `power-domains` prop instead of
  `operating-points-v2` prop, `platform_probe` calls `dev_pm_domain_attach`
  automatically
  - the pmdomain device can build the opp table dynamically for the device as
    well
- `dev_pm_opp_add_dynamic` adds an opp to the device's opp table
  - `opp_table->clk_count` is 0 or 1, depending on `_update_opp_table_clk`
    called when the opp table was allocated
  - `opp_table->regulator_count` is forced to 1, but there is no
    `opp_table->config_regulators`

## OPP Table

- `opp_tables` is a list of all opp tables
- `_find_opp_table` loops through `opp_tables` to find the opp table for a
  device
- `_add_opp_table` allocates an opp table for a device
  - on first call, `_allocate_opp_table` allocates the table
    - `_of_init_opp_table` inits part of the table from OF
      - `operating-points-v2` is the phandle of the table
  - on subsequent calls, `_managed_opp` returns the same table
  - `_update_opp_table_clk` updates the clk for the table
    - it calls `clk_get(dev, NULL)` to get the default clk for the evice
- when driver calls `devm_pm_opp_of_add_table` for its device,
  - `_of_add_table_indexed` calls `_add_opp_table` to add the table
    - it also calls `_of_add_opp_table_v2` to further init the table
      - `_opp_add_static_v2` parses each child node
        - `_opp_allocate` allocates a `dev_pm_opp`
        - `_read_opp_key` parses `opp-hz` and more
        - `opp_parse_supplies` parses `opp-microvolt` and more
- driver can also call `dev_pm_opp_set_config` to add and config the table
  before calling `devm_pm_opp_of_add_table`
  - `config->clk_names` specifies clks explicitly rather than using the
    default clk
  - `config->config_clks` specifies a clk config callback
    - default to `_opp_config_clk_single` if there is 1 clk
  - `config->config_regulators` specifies a regulator config callback
    - default to `_opp_config_regulator_single` if there is 1 regulator
  - `config->regulator_names` specifies regulators explicitly
- when driver calls `dev_pm_opp_set_opp` or `dev_pm_opp_set_rate`, `_set_opp`
  configs for the new opp
  - `_set_required_opps` and `_set_opp_level` call
    `dev_pm_domain_set_performance_state`
  - `_set_opp_bw` calls `icc_set_bw`
  - `config_clks` calls `clk_set_rate`
  - `config_regulators` calls `regulator_set_voltage_triplet`
