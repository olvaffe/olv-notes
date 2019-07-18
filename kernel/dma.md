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

- A scatter list is an array of `struct scatterlist`
  - Each of the `struct scatterlist` in the array describes a (page, offset,
    length) tuple.
  - A `struct scatterlist` can also act as a pointer to another array of
    scatter list.  See `sg_chain`.
  - We like to limit the array size to `SG_MAX_SINGLE_ALLOC` (128) and use
    chaining
- An `sg_table` points to a chain of scatter list arrays
* Say, I have a buffer I want to write to device
  * I call `sg_init_table` on my sg list on stack
  * I manage to make the sg list wrap the buffer.  There are some helper
    functions for this.
  * I queue the sg list to my dma function.

## DMA Mapping

- `get_dma_ops` returns the `dma_map_ops` for a device
  - it uses the device-specific ops if exist
    - on arm, when the device is discovered through DT, `arch_setup_dma_ops`
      is called.  When the device is connected to IOMMU, the call silently
      sets up IOMMU for the device and installs a `dma_map_ops`
  - otherwise, it calls `get_arch_dma_ops(dev->bus)`
    - on x86 it is usually NULL unless there is intel/amd iommu
    - on arm, it is non-NULL
- one can allocate coherent DMA buffer with `dma_alloc_coherent`
  - it is a piece of CPU/device coherent memory
  - `dma_mmap_wc` can map it to user space
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
