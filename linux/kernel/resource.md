Kernel Resource
===============

## `iomem_resource`

- `iomem_resource` is the root resource for io memory
- it is first populated by `setup_arch` called from `start_kernel`
  - on arm64, `request_standard_resources` calls `insert_resource` to insert a
    few resources
    - `Kernel code` resource
    - `Kernel data` resource
    - each memblock becomes a `reserved` or `System RAM` resource
  - on x86, `setup_arch` calls `insert_resource` to insert a few resources
    - `Kernel code` resource
    - `Kernel rodata` resource
    - `Kernel data` resource
    - `Kernel bss` resource
    - `probe_roms` inserts
      - `System ROM` resource
    - `e820__reserve_resources` inserts a resource for each e820 entry
      - `E820_TYPE_RAM` becomes `System RAM` res
      - `E820_TYPE_ACPI` becomes `ACPI Tables` res
      - `E820_TYPE_NVS` becomes `ACPI Non-volatile Storage` res
      - `E820_TYPE_RESERVED` becomes `Reserved` res
      - unaligned regions are padded with the paddings as `RAM buffer` res
- more resources are inserted as the system boots
  - `acpi_parse_hpet` parses `ACPI_SIG_HPET` table to `HPET %u` res
  - `ioapic_setup_resources` inserts `IOAPIC %u` res
  - `pci_mmconfig_alloc` inits `PCI ECAM %04x [bus %02x-%02x]` res
  - `pci_acpi_root_add_resources` inserts `PCI Bus %04x:%02x` res
  - `dmar_parse_one_drhd` calls `request_mem_region` to add `dmar%d` res
  - `system_pnp_probe` calls `request_mem_region` to add `pnp %s` res
  - `intel_pinctrl_probe` calls `devm_platform_ioremap_resource` to add
    `INT34C5:00` res
- `walk_system_ram_range`
  - `vmf_insert_*` calls `track_pfn_insert` which calls `lookup_memtype` which
    calls `pat_pagerange_is_ram` which calls `walk_system_ram_range`
  - `pat_pagerange_is_ram` returns
    - 1 if the range is entirely inside ram
    - 0 if the range is entirely outside ram
    - -1 if the range is partly in ram and partly not in ram
