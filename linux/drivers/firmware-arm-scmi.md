Kernel ARM SCMI
===============

## ARM SCMI

- an soc often has a microcontroller called SCP (System Control Processor)
  - the controller runs on a firmware and supports
    - power domain management,
    - performance management,
    - clock management,
    - sensor management, and
    - reset management
  - the firmware has a vendor-defined protocol and needs a vendor driver to
    talk to SCP
- ARM SCMI, System Control and Management Interface, defines a standard
  protocol for the SCP firmware
  - this allows generic drivers to be written and talk to the firmware
  - it replaces the older SCPI interface

## SCMI FW Driver

- `CONFIG_ARM_SCMI_PROTOCOL`
  - core are `bus.c`, `driver.c`, and `notify.c`
  - protocols are `base.c`, `clock.c`, `perf.c`, `power.c`, `reset.c`,
    `sensors.c`, `system.c`, `voltage.c`, `powercap.c`, and `pinctrl.c`
  - transports are `shmem.c`, `mailbox.c`, `smc.c`, `msg.c`, `virtio.c`, and
    `optee.c`
- `scmi_bus_init` inits the subsystem
- `scmi_driver_init` inits the transports and protocols
  - it also registers a platform driver that matches against `arm,scmi*`
- if the DT defines a compat node, `scmi_probe` probes
  - `scmi_device_create` is called for child nodes

## SCMI FW Consumers

- `CONFIG_COMMON_CLK_SCMI`
  - `module_scmi_driver(scmi_clocks_driver)` registers an scmi driver
    that matches against `SCMI_PROTOCOL_CLOCK`
  - `scmi_clocks_probe` calls `devm_clk_hw_register` to register clks and
    calls `devm_of_clk_add_hw_provider` to register a provider
  - ops in `clk_ops` are forwarded to the scmi firmware
- `CONFIG_ARM_SCMI_CPUFREQ`
  - `module_scmi_driver(scmi_cpufreq_drv)` registers an scmi driver
    that matches against `SCMI_PROTOCOL_PERF`
  - `scmi_cpufreq_probe` calls `cpufreq_register_driver` to register a cpufreq
    driver
  - ops in `cpufreq_driver` are forwarded to the scmi firmware
- `CONFIG_SENSORS_ARM_SCMI`
  - `module_scmi_driver(scmi_hwmon_drv)` registers an scmi driver
    that matches against `SCMI_PROTOCOL_SENSOR`
  - `scmi_hwmon_probe` calls `devm_hwmon_device_register_with_info` to
    register a hwmon dev and calls `devm_thermal_of_zone_register` to register
    thermal zones
  - ops in `hwmon_ops` and `thermal_zone_device_ops` are forwarded to the scmi
    firmware
- `CONFIG_IIO_SCMI`
  - `module_scmi_driver(scmi_iiodev_driver)` registers an scmi driver
    that matches against `SCMI_PROTOCOL_SENSOR`
  - `scmi_iio_dev_probe` calls `devm_iio_device_alloc` to alloc iio devs and
    calls `devm_iio_device_register` to register them
  - ops in `iio_buffer_setup_ops` and `iio_info` are forwarded to the scmi
    firmware
- `CONFIG_PINCTRL_SCMI`
  - `module_scmi_driver(scmi_pinctrl_driver)` registers an scmi driver
    that matches against `SCMI_PROTOCOL_PINCTRL`
  - `scmi_perf_domain_probe` calls `devm_pinctrl_register_and_init` to
    registers a pinctrl and calls `pinctrl_enable` to enable it
  - ops in `pinconf_ops` are forwarded to the scmi firmware
- `CONFIG_ARM_SCMI_PERF_DOMAIN`
  - `module_scmi_driver(scmi_perf_domain_driver)` registers an scmi driver
    that matches against `SCMI_PROTOCOL_PERF`
  - `scmi_perf_domain_probe` calls `pm_genpd_init` to init genpds and calls
    `of_genpd_add_provider_onecell` to register a genpd provider
  - all ops are forwarded to the scmi firmware
    - only `set_performance_state`
- `CONFIG_ARM_SCMI_POWER_DOMAIN`
  - `module_scmi_driver(scmi_power_domain_driver)` registers an scmi driver
    that matches against `SCMI_PROTOCOL_POWER`
  - `scmi_pm_domain_probe` calls `pm_genpd_init` to init genpds and calls
    `of_genpd_add_provider_onecell` to register a genpd provider
  - all ops are forwarded to the scmi firmware
    - only `power_on` and `power_off`
- `CONFIG_ARM_SCMI_POWERCAP`
  - `scmi_powercap_init` registers an scmi driver that matches against
    `SCMI_PROTOCOL_POWERCAP`
  - `scmi_powercap_probe` calls `powercap_register_zone` to register powercap
    zones
  - ops in `powercap_zone_ops` and `powercap_zone_constraint_ops` are
    forwarded to the scmi firmware
- `CONFIG_REGULATOR_ARM_SCMI`
  - `module_scmi_driver(scmi_drv)` registers an scmi driver that matches
    against `SCMI_PROTOCOL_VOLTAGE`
  - `scmi_regulator_probe` calls `devm_regulator_register` to register
    regulators managed by the fw
  - ops in `regulator_ops` are forwarded to the scmi firmware
- `CONFIG_RESET_SCMI`
  - `module_scmi_driver(scmi_reset_driver)` registers an scmi driver that
    matches against `SCMI_PROTOCOL_RESET`
  - `scmi_reset_probe` calls `devm_reset_controller_register` to register a
    reset controller
  - ops in `reset_control_ops` are forwarded to the scmi firmware
