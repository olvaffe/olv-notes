Kernel IOMMU
============

## Core

- config
  - `IOMMU_DEFAULT_DMA_STRICT`
    - this is the default on arm
    - `iommu.passthrough=0 iommu.strict=1`
    - set `iommu_def_domain_type` to `IOMMU_DOMAIN_DMA` and set
      `iommu_dma_strict`
  - `IOMMU_DEFAULT_DMA_LAZY`
    - this is the default on x86
    - `iommu.passthrough=0 iommu.strict=0`
    - set `iommu_def_domain_type` to `IOMMU_DOMAIN_DMA_FQ`
  - `IOMMU_DEFAULT_PASSTHROUGH`
    - `iommu.passthrough=1`
    - set `iommu_def_domain_type` to `IOMMU_DOMAIN_IDENTITY`
- `iommu_subsys_init`

## AMD

- early init
  - `pci_iommu_alloc` is called from x86 `mem_init`
  - `amd_iommu_detect` calls `iommu_go_to_state` to transition from
    `IOMMU_START_STATE` to `IOMMU_IVRS_DETECTED`
    - this calls `detect_ivrs` to detect acpi IVRS table
  - `x86_init.iommu.iommu_init` is set to `amd_iommu_init`
- init
  - `pci_iommu_init` is an initcall that calls `x86_init.iommu.iommu_init`
  - `amd_iommu_init` transitions from `IOMMU_IVRS_DETECTED` to
    `IOMMU_INITIALIZED`
  - `early_amd_iommu_init`
    - `init_iommu_all` calls `init_iommu_one` and `init_iommu_one_late` on
      each iommu
  - `early_enable_iommus` enables all iommus
  - `amd_iommu_init_pci` initializes iommus as pci devices
  - `enable_iommus_vapic`
  - `enable_iommus_v2`
  - `amd_iommu_enable_interrupts`

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
