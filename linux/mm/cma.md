Kernel CMA
==========

## `CONFIG_CMA`

- `CONFIG_CMA` enables CMA support in mm
- `cma_init_reserved_mem` adds a reserved memory region to cma
  - it creates a `struct cma` to manage the region
- `cma_alloc` allocs from a `struct cma`

## `CONFIG_DMA_CMA`

- arch calls `dma_contiguous_reserve` early on to add a reserved memory region
  to cma
  - it inits `dma_contiguous_default_area`
- `CONFIG_OF_RESERVED_MEM`
  - OF
    - a reserved memory region compatible with `shared-dma-pool`
    - a device with `memory-region` referring the reserved memory region
  - `rmem_cma_setup` probes the reserved memory region
    - it calls `cma_init_reserved_mem` to add the reserved memory to cma
    - if the node is marked as `linux,cma-default`,
      `dma_contiguous_default_area` is set to the region
  - when a device driver probes a device with `memory-region`,
    `of_reserved_mem_device_init` finds the reserved memory region and calls
    `rmem_cma_device_init` to init `dev->cma_area`
- `dma_alloc_contiguous` allocs from either `dev->cma_area` or
  `dma_contiguous_default_area`

## `CONFIG_DMABUF_HEAPS_CMA`

- this enables a dma-buf cma heap
- on init, `add_default_cma_heap` creates the heap
  - it requires `dma_contiguous_default_area` to exist
- `cma_heap_allocate` allocs from `dma_contiguous_default_area`
