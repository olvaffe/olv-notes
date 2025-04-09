Kernel OPP
==========

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
