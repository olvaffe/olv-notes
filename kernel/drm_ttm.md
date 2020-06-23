Kernel DRM TTM
==============

## TTM, using amdgpu as an example

- `amdgpu_ttm_init` calls `ttm_bo_device_init` to initialize the
  `ttm_bo_device`
  - `drm_vma_offset_manager` is used to generate mmap offset of the bo for the
    userspace
  - `ttm_bo_init_mm` is callled to initialize several heaps
    - `TTM_PL_SYSTEM` is ttm-internal
    - `TTM_PL_VRAM` is the vram heap
    - `TTM_PL_TT` is the GTT aperture
    - `AMDGPU_PL_GDS`
    - `AMDGPU_PL_GWS`
    - `AMDGPU_PL_OA`
  - the driver is `amdgpu_bo_driver`.  `drv->init_mem_type` is called for each
    memory type.  `amdgpu_init_mem_type` uses different
    `ttm_mem_type_manager_func` for different heaps
- `amdgpu_gem_create_ioctl,`
  - create a `drm_gem_object`, gem bo
  - the gem bo is embedded in a `struct ttm_buffer_object`
  - the ttm bo is embedded in a `struct amdgpu_bo`
  - `amdgpu_bo_do_create` calls `ttm_bo_init_reserved` to initialize the
    `ttm_buffer_object`
    - the size of the buffer is converted to number of pages needed
    - `drm_vma_offset_add` is called to set up the address space in the vma
      manager so that it can be looked up and mapped later in userspace
    - `ttm_bo_validate` is called to validate the ttm bo.  It moves the bo to
      have correct placement (vram, gtt, etc.).  `ttm_bo_handle_move_mem`
      calls `ttm_tt_create` to create the associated `ttm_tt`.  A `ttm_tt` is
      used to hold pages and `sg_table`.  It uses a `amdgpu_backend_func`.
- `amdgpu_gem_mmap_ioctl`
  - returns the mmap offset already set up for the bo
- `amdgpu_mmap`
  - this function is called when the userspace mmap()
  - it calls `ttm_bo_mmap`.  The bo is looked up using the offset.
    `ttm_bo_vm_ops` is installed
  - `ttm_bo_vm_fault`
    - `fault_reserve_notify` points to `amdgpu_bo_fault_reserve_notify`, which
      calls `ttm_bo_validate` with `AMDGPU_GEM_CREATE_CPU_ACCESS_REQUIRED` to
      move the bo to a cpu-mappable location
    - `ttm_mem_io_reserve_vm` calls `amdgpu_ttm_io_mem_reserve` which softpins the
      bo (i.e., reserve a address space range) when the placement is vram
    - for bo on vram, `ttm_bo_io_mem_pfn` returns the pfn for mmio
    - for bo on gtt, `ttm_tt_populate` pins and maps the pages for DMA
    - `vmf_insert_mixed_prot` or `vmf_insert_pfn_prot` to modify the PTE
- `amdgpu_cs_ioctl`
  - `amdgpu_cs_parser_bos` is called to validate the bos involved
    - `ttm_eu_reserve_buffers` locks and reserves the `dma_resv` for each bo
    - `amdgpu_cs_list_validate` calls `ttm_bo_validate` to move the bos to the
      right places
  - `amdgpu_cs_submit` submits the work
    - `ttm_eu_fence_buffer_objects` adds the fence to each bo
- when the buffer needs to be moved, `ttm_bo_move_buffer` is called
  - it calls `ttm_bo_mem_space` to reserve a range in the address space of the
    new memory type
  - it then calls `ttm_bo_handle_move_mem` to bind the bo in the new memory type
    and move the data
