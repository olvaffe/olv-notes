Kernel and DMA
==============

## DMA Mapping

* It is an API a platform could implement, or not.
  * It allows us map a buffer/page/sg to get DMA-able address(es)
  * It also allows us to allocate a buffer that is DMA-able.
  * However, different archs have different means to start the transfer.  Some
    might have no such mean.  The returned DMA-able address might have no use.
* If it is implemented, the API is named like `dma_xxx_xxx`.
* To use it, include `linux/dma-mapping.h`
  * It will include `asm/dma-mapping.h` or `asm-generic/dma-mapping-broken.h`,
    depending on whether the platform has `CONFIG_HAS_DMA`.
* For drivers that include `linux/pci.h`, the header includes `asm/pci.h`,
  which, on x86 or arm, includes `asm-generic/pci-dma-compat.h`, which gives PCI
  version of the DMA Mapping API.

## DMA Mapping: Streaming

* `dma_map_page` and `dma_unmap_page` are used for DMA streaming.
  * Mapping a page involves finding the bus address, and flushing/invalidating
    cpu cache.
  * The bus address is usually `page_to_phys(page)`, which is checked against
    device's dma mask.  It does not guarantee it is DMA'able.
  * CPU should not touch a mapped page.
  * It relies on other means to start the transfer.
  * `dma_unmap_page` should be called only after DMA is done.
* A common scenario is
  * a driver maps a page and pass it to deivce to receive a buffer
  * irq, and the transfer is done.  the driver unmaps the page.
    * `pci_dma_sync_single_for_cpu` and `pci_dma_sync_single_for_device` can be
      used to check if the buffer is complete beforing unmapping.

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

## IOMMU

- when a device is connected to IOMMU on a arch, IOMMU is properly and
  transparently set up through the dma ops returned by `get_arch_dma_ops`
- there is a generic DMA-API to IOMMU-API glue layer,
  `drivers/iommu/dma-iommu.c`, to simplify the implementation of the dma ops

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

## Archs

- Classic ARM with MMU/IOMMU
  - when a dma device is found in the device tree, `of_dma_configure` is
    called.  Depending on whether the device is dma coherent, also specified
    in device tree, `arch_setup_dma_ops` picks `iommu_coherent_ops` or `iommu_ops`.
    - when without iommu, it picks `arm_coherent_dma_ops` or `arm_dma_ops`
  - when device is coherent, `dma_alloc_coherent` allocates with `alloc_pages`
    - the virtual address from the linear map is returned for CPU
    - the pages are mapped in IOMMU and `dma_addr_t` is returned
    - when device is incoherent, the only difference is that the virtual
      address is not from the linear map.  Instead, a UC or WC vmap is set up.
    - because it uses FLATMEM memory model,j:

  - always use `arm_dma_ops`
