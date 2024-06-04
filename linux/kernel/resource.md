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
- device resources
  - e.g., a pci device has a few resources for mmio (and io)
    - they describe the resources but are not a part of `iomem_resource` tree
  - `__request_region` requests a region
    - `alloc_resource` allocates a new `resource`
    - `__request_resource` adds the new resource to the tree
- `walk_system_ram_range`
  - `vmf_insert_*` calls `track_pfn_insert` which calls `lookup_memtype` which
    calls `pat_pagerange_is_ram` which calls `walk_system_ram_range`
  - `pat_pagerange_is_ram` returns
    - 1 if the range is entirely inside ram
    - 0 if the range is entirely outside ram
    - -1 if the range is partly in ram and partly not in ram

## APIs

- `request_resource_conflict` requests a resource from an immediate parent
  - this adds the resource as an immediate child of the parent
  - a future request with an overlapping range will conflict with this one
  - if a conflict happens, the conflicting resource is returned
  - `request_resource` is a wrapper
- `release_resource` releases a resource from its immediate parent
  - it removes the resource from its immediate parent
- `release_child_resources` recursively releases children of a resource
- `insert_resource_conflict` inserts a request to an immediate parent
  - if there is no conflict, it is the same as `request_resource_conflict`
  - if there is a conflict, and the conflicting resource is a subset, this new
    resource replaces the conflicting resource, and the conflicting resource
    is reparented to this new resource
  - if a real conflict happens, the conflicting resource is returned
- `remove_resource` removes a resource from its immediate parent
  - if the resource has children, the children are reparented
- `allocate_resource` requests a resource of flexible region from an immediate
  parent
  - it determines the final region based on the specified constraints
  - it then calls `__request_resource`
- `lookup_resource` finds the resource with matching start from an immediate
  parent
- `adjust_resource` updates the region of a resource
  - it checks for conflicts before updating
- `resource_alignment` returns the alignment
  - `IORESOURCE_SIZEALIGN` uses the size as alignment
  - `IORESOURCE_STARTALIGN` uses the start as alignment
- `__request_region` requests a region from a parent
  - it `kzalloc` the resource
  - if it conflicts with a sibling, and the sibling is not `IORESOURCE_BUSY`,
    it retries using the sibling as the parent
- `__release_region` releases a region from a parent
  - it find the resource corresponding to the region from children and
    grandchildren
  - it `kfree` the resource
- `request_free_mem_region` is similar to `__request_region` except the region
  is flexible
  - it determines the final region based on the specified constraints
- `alloc_free_mem_region` is similar to `allocate_resource`, except the
  resource is `kalloc`ed
- `walk_*`
