Kernel Powercap
===============

## RAPL

- `/sys/devices/virtual/powercap` has two control types, `intel-rapl` and
  `intel-rapl-mmio`
- `intel-rapl` has two power zones
  - `intel-rapl:0` for `package-0`
    - `intel-rapl:0:0` for `core`
    - `intel-rapl:0:1` for `uncore`
  - `intel-rapl:1` for `psys`
- `intel-rapl-mmio` has one power zone
  - `intel-rapl-mmio:0` for `package-0`

## Core

- `powercap_register_control_type` registers a control type
- `powercap_register_zone` registers a power zone
