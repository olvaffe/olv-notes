Linux MFD syscon
================

## Overview

- syscon stands for system controller
  - it is a group of system registers that do not belong to any driver
  - there are usually several syscon devices in the DT
- `syscon` platform driver
  - it matches the platform device that is named `syscon` exactly
  - it does not match OF node with `syscon` in the compat string
  - as such, it is rarely used

## `syscon_node_to_regmap`

- `syscon_node_to_regmap` early returns unless the node has `syscon` in the
  compat string
- `of_syscon_register` registers a record for the node in `syscon_list`
  - `regmap_init_mmio` returns a `regmap`
