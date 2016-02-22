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

* A scatter list is an array of `struct scatterlist`
  * Each of the elements describes a (page, offset, length) pair.
* Say, I have a buffer I want to write to device
  * I call `sg_init_table` on my sg list on stack
  * I manage to make the sg list wrap the buffer.  There are some helper
    functions for this.
  * I queue the sg list to my dma function.
