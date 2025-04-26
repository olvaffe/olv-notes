Kernel SoundWire
================

## Hardware Interfaces

- mobile
  - I2S, 1986
  - SLIMbus, 2007
- pc
  - AC97, 1997
  - HDA, 2004
- SoundWire, 2014

## Overview

- controller driver
  - `sdw_bus_master_add` adds a master during probe
    - `sdw_of_find_slaves` calls `sdw_slave_add` to add slaves from child
      nodes
      - the compat string is `sdw<ver><mfg_id><part_id><class_id>`
  - `sdw_slave_add` adds a slave when the master detects one as well
- codec driver
  - `module_sdw_driver` registers a `sdw_driver`
  - `snd_soc_register_component` registers an asoc component when the driver
    probes a slave
