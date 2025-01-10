Kernel Powercap
===============

## RAPL

- `/sys/devices/virtual/powercap` has two control types, `intel-rapl` and
  `intel-rapl-mmio`
- `intel-rapl` has two power zones
  - `intel-rapl:0` for `package-0`
    - `intel-rapl:0:0` for `core`
    - `intel-rapl:0:1` for `uncore`, if igpu
    - `intel-rapl:0:1` for `dram`, if server or old
  - `intel-rapl:1` for `psys`
- `intel-rapl-mmio` has one power zone
  - `intel-rapl-mmio:0` for `package-0`
- power zones
  - `core` measures cpu cores
  - `uncore` measures mainly igpu
  - `dram` measures dram
  - `package` measures `core`, `uncore`, l3, memory controller, etc.
    - but not `dram`
  - `psys` measures the entire system
    - what a "system" means is platform-defined
    - on a laptop, it typically means the entire laptop including the screen
      (that is, it measures the power input)
- `sudo turbostat -q -i 1 -S -s Avg_MHz,GFXMHz,PkgWatt,CorWatt,GFXWatt,RAMWatt,SysWatt`


## Core

- `powercap_register_control_type` registers a control type
- `powercap_register_zone` registers a power zone
