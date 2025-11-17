Kernel SPMI
===========

## Spec

- SPMI is specified by MIPI
  - System Power Management Interface
  - 2008, 1.0
  - 2012, 2.0
- devices on SPMI are typically PMICs

## Controller Driver

- `devm_spmi_controller_alloc` allocs a controller
- `spmi_controller_add` adds the controller to the core
  - it automatically calls `of_spmi_register_devices` to populate the bus

## PMIC Driver

- `module_spmi_driver` calls `spmi_driver_register` to register a driver
