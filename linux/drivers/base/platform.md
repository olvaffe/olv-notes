Kernel platform device
======================

## Platform Bus

- `platform_bus_init`
  - `device_register` registers `platform_bus`, for `/sys/devices/platform`
  - `bus_register` registers `platform_bus_type` for platform devices
- `platform_match`
  - `of_driver_match_device` matches compat against `drv->of_match_table`
  - `acpi_driver_match_device` matches pnp id against `drv->acpi_match_table`
  - `platform_match_id` matches `pdev->name` against `pdrv->id_table`
  - finally, it matches `pdev->name` against `drv->name`
- `platform_dma_configure`
  - `of_dma_configure` applies dma config from dt node
    - `dev->dma_mask` and `dev->coherent_dma_mask` are updated
    - `of_iommu_configure` applies iommu config from dt node
      - `of_iommu_configure_device`
        - `iommu_fwspec_init`
          - `dev_iommu_get` allocs `dev->iommu`
          - `dev_iommu_fwspec_set` sets `dev->iommu->fwspec`
        - if smmu, `arm_smmu_of_xlate`
      - `iommu_probe_device`
        - `iommu_init_device`
        - `iommu_setup_default_domain`
        - `iommu_setup_dma_ops`
  - `acpi_dma_configure`
  - `iommu_device_use_default_domain` increments `dev->iommu_group->owner_cnt`
- `platform_probe`
  - `of_clk_set_defaults` parses `assigned-clock-parents` prop
  - `dev_pm_domain_attach`
    - `genpd_dev_pm_attach` parses `power-comains` prop
- comebined with `really_probe`, these are called
  - `device_links_check_suppliers` checks that all suppliers are
    `DL_STATE_AVAILABLE`
  - `pinctrl_bind_pins`
  - `dev->bus->dma_configure` is `platform_dma_configure`
  - `dev->pm_domain->activate`
  - `dev->bus->probe` is `platform_probe`
