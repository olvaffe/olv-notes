Kernel PM Domain
================

## Controllers

- the pmdomain subsystem was renamed from genpd, Generic PM Domain
- a power controller has many power rails
  - each rail is called a pm domain
  - different groups of devices are attached to different rails
  - when all devices are on a rail are idle, the rail can be gated
- dt node, `power: power-controller@12340000`
  - `#power-domain-cells = <1>;` means each consuming device uses a cell (u32)
    to identify the rail
  - children nodes to identify different pm domains
  - there may be grandchildren as well for subdomains
- driver initialization
  - driver allocates a `struct generic_pm_domain *genpd` for each pm domain
    and calls `pm_genpd_init` to add the domain to the core
  - driver calls `of_genpd_add_provider_onecell` to add the controller to the
    core, as a pm domain provider

## Consumers

- dt node, `consumer@12350000`
  - `power-domains = <&power 0>;` means it uses rail `0`
- the consumer driver can call `dev_pm_domain_attach_by_name` to attach the
  consumer device to a named pm domain
  - the bus, such as `platform_probe`, also calls `dev_pm_domain_attach` for
    the common case where the device has only a single power domain
  - `power-domain-names` is parsed to get an index
  - `__genpd_dev_pm_attach` parses `power-domains` and calls
    `genpd_get_from_provider` to find the provider
  - `genpd_add_device` adds the device to the genpd
