Kernel and DMA
==============

## Configs

- every source file is optional
  - `obj-$(CONFIG_HAS_DMA)                   += mapping.o direct.o`
    - arm and x86
  - `obj-$(CONFIG_DMA_OPS_HELPERS)           += ops_helpers.o`
    - arm and x86
  - `obj-$(CONFIG_ARCH_HAS_DMA_OPS)          += dummy.o`
    - neither arm nor x86 by default
  - `obj-$(CONFIG_DMA_CMA)                   += contiguous.o`
    - optional
  - `obj-$(CONFIG_DMA_DECLARE_COHERENT)      += coherent.o`
    - arm but not x86
    - `CONFIG_OF_RESERVED_MEM`
  - `obj-$(CONFIG_SWIOTLB)                   += swiotlb.o`
    - arm and x86
  - `obj-$(CONFIG_DMA_COHERENT_POOL)         += pool.o`
    - arm but not x86
  - `obj-$(CONFIG_MMU)                       += remap.o`
    - arm and x86
    - used by `CONFIG_IOMMU_DMA` to translate dma-api to iommu-api

## `CONFIG_OF_RESERVED_MEM`

- device tree
  - `/reserved_memory` has a child node for each reserved regions
  - device nodes have `memory-region` to refer to the reserved regions
- `RESERVEDMEM_OF_DECLARE` declares an init func for a dt node
  - `CONFIG_DMA_RESTRICTED_POOL` calls `rmem_swiotlb_setup` for
    `restricted-dma-pool`
  - `CONFIG_DMA_DECLARE_COHERENT` calls `rmem_dma_setup` for `shared-dma-pool`
  - `CONFIG_DMA_CMA` calls `rmem_cma_setup` for `shared-dma-pool`
  - they are added to `__reservedmem_of_table` array
- `fdt_scan_reserved_mem_reg_nodes` parses `/reserved-memory`
  - for each child node, it finds the matching init function from
    `__reservedmem_of_table` array to init the region
- when a device driver probes a device node, and the device node has
  `memory-region` to refer to reserved regions, the driver calls
  `of_reserved_mem_device_init`
- `rmem_dma_setup` requires the rmem to have no `reusable`
  - when `rmem_dma_device_init` is called,
    - `dma_init_coherent_memory` wraps the rmem in a `dma_coherent_mem`
    - `dma_assign_coherent_memory` assigns the mem to `dev->dma_mem`
- `rmem_cma_setup` requires the rmem to have `reusable`
  - `cma_init_reserved_mem` wraps the rmem in a `cma` and adds the cma to
    `cma_areas` array
  -  when `rmem_cma_device_init` is called, `dev->cma_area` is set to the cma
     area

## `CONFIG_IOMMU_DMA`

- it translates dma-api to iommu-api
- the iommu subsys calls `iommu_probe_device` on devices behind iommu
  - e.g., for platform devices, `platform_dma_configure` calls
    `of_dma_configure` which calls `of_iommu_configure`
- `iommu_setup_dma_ops`
  - `dev->dma_iommu` is set if the iommu domain has `IOMMU_DOMAIN_DMA` or
    `IOMMU_DOMAIN_DMA_FQ`

## Arch

- before a device is to be probed by a driver, the bus system calls the
  optional `dma_configure` op on the device
  - pci and platform busses have the op
    - `pci_bus_type` and `pci_dma_configure`
    - `platform_bus_type` and `platform_dma_configure`
  - at the end of the op, `arch_setup_dma_ops` is called to set up device dma
    ops
    - x86: NULL
    - arm64: NULL
    - arm: picks when the device is behind an iommu, pick `iommu_coherent_ops`
      or `iommu_ops`; otherwise, pick `arm_coherent_dma_ops` or `arm_dma_ops`
- when `dma_map_sg` is called, `get_dma_ops` returns the device's dma ops or
  a global one returned by `get_arch_dma_ops`
  - x86: NULL
  - arm64: NULL
  - arm: `arm_dma_ops` or NULL
- Classic ARM with MMU/IOMMU
  - when device is coherent, `dma_alloc_coherent` allocates with `alloc_pages`
    - the virtual address from the linear map is returned for CPU
    - the pages are mapped in IOMMU and `dma_addr_t` is returned
    - when device is incoherent, the only difference is that the virtual
      address is not from the linear map.  Instead, a UC or WC vmap is set up.
    - because it uses FLATMEM memory model,j:

## DMA Mapping

- It is an API a platform could implement, or not.
  - It allows us map a buffer/page/sg to get DMA-able address(es)
  - It also allows us to allocate a buffer that is DMA-able.
  - However, different archs have different means to start the transfer.  Some
    might have no such mean.  The returned DMA-able address might have no use.
- If it is implemented, the API is named like `dma_xxx_xxx`.
- To use it, include `linux/dma-mapping.h`
  - It will include `asm/dma-mapping.h` or `asm-generic/dma-mapping-broken.h`,
    depending on whether the platform has `CONFIG_HAS_DMA`.
- For drivers that include `linux/pci.h`, the header includes `asm/pci.h`,
  which, on x86 or arm, includes `asm-generic/pci-dma-compat.h`, which gives PCI
  version of the DMA Mapping API.

## DMA Mapping: Streaming

- `dma_map_page` and `dma_unmap_page` are used for DMA streaming.
  - Mapping a page involves finding the bus address, and flushing/invalidating
    cpu cache.
  - The bus address is usually `page_to_phys(page)`, which is checked against
    device's dma mask.  It does not guarantee it is DMA'able.
  - CPU should not touch a mapped page.
  - It relies on other means to start the transfer.
  - `dma_unmap_page` should be called only after DMA is done.
- A common scenario is
  - a driver maps a page and pass it to deivce to receive a buffer
  - irq, and the transfer is done.  the driver unmaps the page.
    - `pci_dma_sync_single_for_cpu` and `pci_dma_sync_single_for_device` can be
      used to check if the buffer is complete beforing unmapping.
- for a single dma tranaction, we can
  - `dma_map_page(dir)`
  - start hw transction
  - wait hw transction
  - `dma_unmap_page(dir)`
  - `dir` must be the same for map and unmap
- for multiple dma tranactions, we can
  - `dma_map_page(dir)`
  - loop
    - `dma_sync_single_for_device(dir)`
      - can be skipped when `dir` is `DMA_FROM_DEVICE`, because there is no
        cpu write
    - start hw transction
    - wait hw transction
    - `dma_sync_single_for_cpu(dir)`
  - `dma_unmap_page(dir)`
  - `dir` must be the same for all calls

## Scatter-Gather

- A `struct scatterlist` describes a contiguous memory range
  - `page_link` is the starting page
  - `offset` is the offset in the starting page
  - `length` is the length of the range and can exceed the starting page
- A `struct sg_table` manages `struct scatterlist` nodes
  - `sg_alloc_table` allocates the specified count of nodes
  - when the number of nodes exceeds `SG_MAX_SINGLE_ALLOC` (128),
    `sg_alloc_table` makes multiple memory allocations for the nodes.
    - A `struct scatterlist` can also act as a pointer to another array of
      scatter list.  See `sg_chain`.
- take `sg_alloc_table_from_pages` for example
  - `[offset, size]` is the data range in the discontiguous buffer described
    as a list of pages
  - to set the `sg_table` up, the number of chunks is decided first
    - when the discontiguous buffer is contiguous, the number of chunks is 1
  - `sg_alloc_table` is called with the number of chunks
    - unless there are more chunks than `SG_MAX_SINGLE_ALLOC`, it allocates an
      array of scatterlist
    - when it needs more than `SG_MAX_SINGLE_ALLOC` chunks, the last element's
      `page_link` points to the next array of scatterlist
  - `sg_set_page` is called on each scallterlist to initialize `page_link` to
    point to the starting page of the chunk

## DMA Mapping

- `get_dma_ops` returns the `dma_map_ops` for a device
  - it uses the device-specific ops if exist
    - on arm, when the device is discovered through DT, `arch_setup_dma_ops`
      is called.  When the device is connected to IOMMU, the call silently
      sets up IOMMU for the device and installs a `dma_map_ops`
  - otherwise, it calls `get_arch_dma_ops(dev->bus)`
    - on x86 it is usually NULL unless there is intel/amd iommu
    - on arm, it is non-NULL
- one can allocate coherent DMA buffer with `dma_alloc_coherent` (or
  `dma_alloc_wc`)
  - it is a piece of CPU/device coherent memory
  - it returns both a virtual address for CPU and a `dma_addr_t`
  - `dma_mmap_coherent` (or `dma_mmap_wc`) can map it to user space
- oen can also use the streaming API `dma_map_sg` and families with regular
  pages
  - it syncs for device on `dma_map`, and syncs for CPU on `dma_unmap`
    - no-op on x86 and arm
    - invalidate/flush CPU caches on arm64
  - it can also be persistently mapped; then explicit flush/invalidate with
    `dma_sync_sg_for_device`/`dma_sync_sg_for_cpu` is required

## swiotlb

- 64-bit kernels with >4GB RAM normally require swiotlb
- on x86-64
  - dmesg shows
    - `PCI-DMA: Using software bounce buffering for IO (SWIOTLB)`
    - `software IO TLB: mapped [mem 0x7bfdf000-0x7ffdf000] (64MB)`
  - there is a 16MB ISA DMA zone, `MAX_DMA_PFN`
  - there is a 4GB DMA32 zone, `MAX_DMA32_PFN`
  - `pci_swiotlb_detect_4gb` detects if there is more than 4GB of memory and
    enables swiotlb
- direct dma (no `dma_map_ops`) falls back to swiotlb transparently
  - swiotlb allocates 64MB of memory pool from memblock
  - when a device cannot DMA-access a page, `dma_map_sg` falls back to
    `swiotlb_tbl_map_single`.  swiotlb allocates a bounce (shadow) page from
    its pool and return the dma address to the shadow page 
  - in `dma_unmap_sg` or `dma_sync_sg_for_*`, swiotlb transparently memcpy's
    between the real page and the shadow page

## Allocation

- device props
  - if `CONFIG_ARCH_HAS_DMA_OPS`, `get_dma_ops` returns `dev->dma_ops` or
    `get_arch_dma_ops`
    - `set_dma_ops` is barely called
    - `get_arch_dma_ops` typically returns null
    - this almost always returns null
  - if `CONFIG_IOMMU_DMA`, `use_dma_iommu` returns `dev->dma_iommu`
    - `iommu_probe_device` calls `iommu_setup_dma_ops` if there is iommu
    - because `iommu_def_domain_type` is usually `IOMMU_DOMAIN_DMA`,
      `dev->dma_iommu` is true
    - `dma_alloc_direct` usually returns false as a result when the device is
      behind an iommu
  - `dma_set_mask` sets `dev->dma_mask`
  - `dma_set_coherent_mask` sets `dev->coherent_dma_mask`
- alloc wrappers
  - `dma_alloc_coherent` calls `dma_alloc_attrs`
  - `dma_alloc_wc` calls `dma_alloc_attrs`
  - `dma_alloc_noncoherent` calls `dma_alloc_pages`
- `dma_alloc_wc` calls `dma_alloc_attrs` with `DMA_ATTR_WRITE_COMBINE`
  - `get_dma_ops` returns `NULL` on modern archs, because they don't have
    `CONFIG_ARCH_HAS_DMA_OPS`
  - `dma_alloc_from_dev_coherent` suballocates from `dev->dma_mem`, if any
    - that is a reserved region in `/reserved_memory`
  - `dma_alloc_direct` and `use_dma_iommu`
    - they pick `dma_direct_alloc` or `iommu_dma_alloc` depending on whether
      `dev->dma_iommu` is true
  - `dma_direct_alloc`
    - it seems to allocate contiguous pages
  - `iommu_dma_alloc` allocs through `CONFIG_IOMMU_DMA`
    - it seems to allocate non-contiguous pages
- when using dma iommu,
  - `dma_alloc_pages` calls `dma_common_alloc_pages`
    - it tries `dma_alloc_contiguous` to alloc from cma first
      - otherwise, it `alloc_pages_node` which likely fail for large allocs
    - `iommu_dma_map_page` sets up the iommu
  - `dma_alloc_attrs` calls `iommu_dma_alloc`
    - `iommu_dma_alloc_remap` allocs non-contiguous pages and sets up the
      iommu to have a linear mapping
  - `dma_alloc_noncontiguous` calls `iommu_dma_alloc_noncontiguous` to alloc
    non-contiguous pages and sets up the iommu to have a linear mapping
- when using direct dma
  - `dma_alloc_pages` calls `dma_direct_alloc_pages`
  - `dma_alloc_attrs` calls `dma_direct_alloc`
  - `dma_alloc_noncontiguous` calls `dma_direct_alloc_pages`
