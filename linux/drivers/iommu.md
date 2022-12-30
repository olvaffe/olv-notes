IOMMU
=====

## How to Use

- a driver wants to allocates pages dynamically for DMA by the device
  - e.g., GPU and GEM BOs
- it defines an iova address space, a range of addresses
  - whenever it allocates a DMA-able system memory, it carves out a range from
    the iova space
- it calls `iommu_domain_alloc` with the bus type of the device, and sets the
  iova space to the domain
  - also calls `iommu_set_fault_handler` to set a fault handler to dump
    invalid accesses
- it calls `iommu_attach_device` to attach the device to the IOMMU.  This
  usually sets up a context for the device in the IOMMU and creates a
  `io_pgtable_ops` for manipulating the page table
- given a system memory, it gets the `sg_table` for the system memory.  It
  calls `iommu_map_sg` so that IOMMU can set up iova-to-physical mappings for
  the device
  - each mapping can be a combination of `IOMMU_READ`, `IOMMU_WRITE`,
    `IOMMU_CACHE`, `IOMMU_NOEXEC`, and `IOMMU_MMIO`
  - `IOMMU_CACHE` enables DMA coherency, between the device and the CPU,
    usually by enabling snooping
  - The device uses "bus address space" for memory access, which is also known
    as the "dma address space".  Without IOMMU, "buss address space" is mapped
    to "physical address space" trivially, ofthen with an offset.  With IOMMU,
    it maps "bus address space" to "physical address space" using page tables
