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

## Example

- `mediatek/mt8195.dtsi`
  - `scpsys: syscon@10006000`
    - `compatible = "mediatek,mt8195-scpsys", "syscon", "simple-mfd";`
    - `spm: power-controller`
      - `compatible = "mediatek,mt8195-power-controller";`
      - `mfg0: power-domain@MT8195_POWER_DOMAIN_MFG0`
        - `mfg1: power-domain@MT8195_POWER_DOMAIN_MFG1`
          - `clocks = <&apmixedsys CLK_APMIXED_MFGPLL>, <&topckgen CLK_TOP_MFG_CORE_TMP>;`
          - `clock-names = "mfg", "alt";`
          - `mediatek,infracfg = <&infracfg_ao>;`
          - `power-domain@MT8195_POWER_DOMAIN_MFG2`
- there is no driver bound to `scpsys`
  - there is a `syscon` mfd platform driver, but it is rarely used
  - its helpers such as `syscon_node_to_regmap` are used instead
- `CONFIG_MTK_SCPSYS_PM_DOMAINS` binds to `spm`
  - `mt8195_scpsys_data` is the pm domain definitions for mt8195
  - `syscon_node_to_regmap` is called on `scpsys` to get the regmap
  - `for_each_available_child_of_node` loops through all immediate child nodes
    - `scpsys_add_one_domain` allocs and inits a `scpsys_domain`, which
      contains a `genpd`
    - `scpsys_add_subdomain` loops through all grandchild nodes recursively
      - it calls `scpsys_add_one_domain` on each grandchild node
      - it also calls `pm_genpd_add_subdomain` to build a genpd tree
  - `of_genpd_add_provider_onecell` registers the provider
- when a consumer calls `dev_pm_domain_attach_by_name`
  - `genpd_power_on` is called on all parent pm domains and the specified pm
    domain
  - `scpsys_power_on`
    - `scpsys_regulator_enable` enables regulator, if any
    - `clk_bulk_prepare_enable` preps and enables clks, if any
    - writes regs using regmap
    - `scpsys_sram_enable`
    - `scpsys_bus_protect_disable`
