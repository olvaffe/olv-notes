Kernel OPP
==========

## OPP Table

- drivers call `devm_pm_opp_set_regulators` to set regulators
  - `_add_opp_table` adds the opp table from dt
    - `_allocate_opp_table` allocates the table
      - `_of_init_opp_table` parses `operating-points-v2` to get the phandle
  - `_opp_set_regulators` gets the regulators
- drivers call `devm_pm_opp_of_add_table` to add the opp table from dt
  - `_add_opp_table_indexed` returns the already-added opp table
  - `_of_add_opp_table_v2` parses the opp table from dt
    - `_opp_add_static_v2` parses each child node
      - `_opp_allocate` allocates a `dev_pm_opp`
      - `_read_opp_key` parses `opp-hz` and more
      - `opp_parse_supplies` parses `opp-microvolt` and more
