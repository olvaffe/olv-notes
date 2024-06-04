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

## `struct resource`

- `start` and `end` are the range, both inclusive
- `name` is the descriptive name
- `flags` are resource flags
  - bit 0..7 are `IORESOURCE_BITS` and are subsystem-specific
    - PnP IRQ, DMA, MEM, IO
    - PCI
  - bit 8..12 are `IORESOURCE_TYPE_BITS`
    - `IORESOURCE_IO` is io ports
    - `IORESOURCE_MEM` is mem addrs
    - `IORESOURCE_REG` is regs
    - `IORESOURCE_IRQ` is irqs
    - `IORESOURCE_DMA` is dma ids
    - `IORESOURCE_BUS` is bus ids
  - bit 13 is `IORESOURCE_PREFETCH`
    - it indicates the resource can be accessed with prefetch enabled (because
      there is no side-effect)
    - e.g.., the resource can be mapped with `ioremap_wc` instead of `ioremap`
  - bit 14 is `IORESOURCE_READONLY`
    - it indicates the resource is read-only
  - bit 15 is `IORESOURCE_CACHEABLE` and is pnp-specific
  - bit 16 is `IORESOURCE_RANGELENGTH` and is pnp-specific
  - bit 17 is `IORESOURCE_SHADOWABLE` and is pnp-specific
  - bit 18 is `IORESOURCE_SIZEALIGN`
    - `resource_alignment` returns `resource_size`
  - bit 19 is `IORESOURCE_STARTALIGN`
    - `resource_alignment` returns `start`
  - bit 20 is `IORESOURCE_MEM_64`
    - the resource is beyond 4GB
  - bit 21 is `IORESOURCE_WINDOW`
    - the resource is reserved by a pci bridge?
  - bit 22 is `IORESOURCE_MUXED`
    - it indicates the resource is sw-muxed
    - two drivers might access the same io port
    - they should enclose their accesses by `request_muxed_region` and
      `__release_region` to coordinate
  - bit 23 is unused
  - bit 24 is `IORESOURCE_EXT_TYPE_BITS`
    - this is a modifier for `IORESOURCE_TYPE_BITS`
    - when the type is `IORESOURCE_MEM`, this bit is called
      `IORESOURCE_SYSRAM` and indicates system ram
  - bit 25 is `IORESOURCE_SYSRAM_DRIVER_MANAGED`
  - bit 26 is `IORESOURCE_SYSRAM_MERGEABLE`
  - bit 27 is `IORESOURCE_EXCLUSIVE`
    - it indicates that the resource cannot be accessed from userspace (via
      `/dev/mem` or sysfs nodes)
  - bit 28 is `IORESOURCE_DISABLED`
    - it indicates that the resource is disabled/invalid and should be ignored
  - bit 29 is `IORESOURCE_UNSET`
    - it indicates that start/end has not finalized yet
  - bit 30 is `IORESOURCE_AUTO`
    - this is only used by pnp
  - bit 31 is `IORESOURCE_BUSY`
    - this is mainly used by `request_region`
    - if `request_region` conflicts with the resource, and the resource is not
      busy, `request_region` requests a child of the resource instead
- `desc` is used by `walk_iomem_res_desc` and `region_intersects`
  - it is used to filter by descriptor types, such as `IORES_DESC_ACPI_TABLES`
- `parent`, `sibling`, and `child` are used to form a resource tree
  - `parent` points to the parent, if any
    - the range of the resource is a subset of the range of the parent
  - `sibling` points to the next sibling, if any
    - the range of the resource is before the range of the sibling
  - `child` points to the first child, if any
    - the range of the resource is a superset of the ranges of all children
