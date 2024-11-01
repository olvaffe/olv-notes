Kernel IOMMU
============

## Core

- config
  - `IOMMU_DEFAULT_DMA_STRICT`
    - this is the default on arm
    - `iommu.passthrough=0 iommu.strict=1`
    - set `iommu_def_domain_type` to `IOMMU_DOMAIN_DMA` and set
      `iommu_dma_strict`
    - devices are translated, restricted to dma-mapped pages
    - higher overhead, higher security
  - `IOMMU_DEFAULT_DMA_LAZY`
    - this is the default on x86
    - `iommu.passthrough=0 iommu.strict=0`
    - set `iommu_def_domain_type` to `IOMMU_DOMAIN_DMA_FQ`
  - `IOMMU_DEFAULT_PASSTHROUGH`
    - `iommu.passthrough=1`
    - set `iommu_def_domain_type` to `IOMMU_DOMAIN_IDENTITY`
    - trusted devices are identically-mapped
    - lower overhead, lower security

## Controllers

- <https://www.kernel.org/doc/Documentation/devicetree/bindings/iommu/iommu.txt>
  - a hw uses "bus address space" for memory access, which is also known as
    the "dma address space".  Without iommu, "bus address space" is mapped to
    "physical address space" trivially, often with an offset.  With iommu, it
    maps "bus address space" to "physical address space" using page tables
  - consumers of the iommu controller are called masters
- `iommu_device_register` is called from the probe function of the controller
  driver
  - this adds the iommu to `iommu_device_list`
  - this also calls `bus_iommu_probe` on all known busses, to probe consumer
    devices

## Consumers

- before a consumer device is probed by its driver, `iommu_probe_device`
  should have been called
  - `iommu_subsys_init` registers `iommu_bus_notifier` to call
    `iommu_probe_device` on all devices
  - when a controller driver calls `iommu_device_register`, `bus_iommu_probe`
    probes all devices on all busses
  - when dd calls `driver_probe_device` on a device/driver pair, it calls
    `bus->dma_configure` first
    - on platform bus, `platform_dma_configure` calls `of_dma_configure` which
      calls `of_iommu_configure`
    - on pci bus, `pci_dma_configure` behaves similarly
    - `of_iommu_configure`
      - looks for `iommus` or `iommu-map` OF props in the consumer device
      - calls `iommu_fwspec_init` on the consumer device
      - calls `iommu_ops_from_fwnode` to get ops from the controller
      - calls `ops->of_xlate` on the consumer device
      - calls `iommu_probe_device`
  - `iommu_probe_device` uses these iommu ops
    - `ops->probe_device`
    - `ops->device_group`
    - `ops->is_attach_deferred`
    - `ops->probe_finalize`
- `iommu_domain_alloc` allocates a `iommu_domain`
  - this loops through all devices on the bus and expects all devices share
    the same iommu controller
    - this has been deprecated
  - `ops->domain_alloc`
- `iommu_set_fault_handler` sets the fault handler
  - it is invoked when iommu detects page faults
- `iommu_attach_device` attaches the iommu domain to the consumer device
  - this calls down to `__iommu_attach_device` which calls
    `domain->ops->attach_dev`
  - `attach_dev` often calls `alloc_io_pgtable_ops` to creates a
    `io_pgtable_ops` to manage a page table tree
- `iommu_map` maps a physical addr into the iommu addr space
  - this calls down to `domain->ops->map_pages`
  - when the consumer accesses `iova`, `iova` gets translated to `paddr`
  - the consumer driver is responsible for tracking and finding an unsued
    iova range for mapping
  - `prot`
    - `IOMMU_READ` enables read access
    - `IOMMU_WRITE` enables write access
    - `IOMMU_CACHE` enables DMA coherency, between the device and the CPU,
      usually by enabling snooping
    - `IOMMU_NOEXEC` disables execution
    - `IOMMU_MMIO` enables mmio access
  - `gfp`
    - when mapping, the controller needs to set up page tables
    - page tables are backed by pages allocated with `gfp`
- `iommu_map_sg` maps a scatterlist into the iommu addr space
  - when the consumer accesses the contiguous iova range, it gets mapped to
    a scatterlist

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

## GPU IOMMUs

- some gpus have their own iommu controllers
- msm
  - a2xx uses `MSM_MMU_GPUMMU`
    - it appears to be an ad-hoc simple iommu
    - it does not use the iommu subsystem
  - a3xx to a5xx use `MSM_MMU_IOMMU`
    - it is arm smmu-v2
    - `iommu_paging_domain_alloc`
    - `iommu_attach_device`
    - `iommu_set_fault_handler`
  - a6xx+ use both `MSM_MMU_IOMMU` and `MSM_MMU_IOMMU_PAGETABLE`
    - `adreno_iommu_create_address_space` creates the global address using
      `MSM_MMU_IOMMU`
    - `msm_gpu_create_private_address_space` creates a private address space
      for each gpu context using `MSM_MMU_IOMMU_PAGETABLE`
      - `alloc_io_pgtable_ops(ARM_64_LPAE_S1)` to manage its own page tables
      - `a6xx_set_pagetable` switches the address spaces
- panfrost
  - `panfrost_mmu_ctx_create` creates an mmu context for each gpu context
    - `mmu->mm` is a `drm_mm` and manages a `[32MB, 4GB]` space
    - `mmu->pgtbl_cfg` is a `io_pgtable_cfg`
    - `mmu->pgtbl_ops` is a `ARM_MALI_LPAE` pgtable
      - the iommu is similar to a standard arm smmu
  - `panfrost_mmu_enable` enables the address space
    - there can be multiple enabled ASs and each gpu job can select an AS
  - `panfrost_mmu_map` maps a bo into the AS
    - `drm_mm_insert_node_generic` has been called to find an unused range
    - `mmu->pgtbl_ops->map_pages` sets up the page tables
- panthor
  - a vm has 3 ranges
    - user va range at the bottom, managed entirely by the userspace
    - kernel va range at the top, managed entirely by the kernel space
    - kernel auto va range, a subset of kernel va range managed by `drm_mm`
  - allocations and mappings are separated
    - mapping goes through `drm_gpuvm`, which tracks ranges, bos, and fences
    - `drm_gpuvm` calls down into the driver to do the real map/unmap
  - `panthor_vm_create` creates a vm
    - `alloc_io_pgtable_ops(ARM_64_LPAE_S1)`
  - `panthor_vm_active` enables a vm
    - a job refers to the vm using AS id returned by `panthor_vm_as`
  - `panthor_vm_map_pages` maps a sgt into the vm
    - `vm->pgtbl_ops->map_pages` sets up the page tables
