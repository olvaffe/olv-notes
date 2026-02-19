DRM XE
======

## Initialization

- `xe_irq_install` installs irq
  - `guc2host_irq_handler`
  - `xe_irq_msix_default_hwe_handler`
- `xe_gt_init` inits a GT
  - `xe_execlist_init` is nop if uc is enabled
  - `xe_hw_engines_init` inits engines
  - `xe_uc_init_post_hwconfig`
    - `xe_guc_init_post_hwconfig` calls `xe_guc_submit_init`
      - `gt->exec_queue_ops = &guc_exec_queue_ops;`

## IOCTLs

- `xe_exec_queue_create_ioctl` creates a queue
  - `xe_hw_engine_lookup` looks up the engine
  - `xe_exec_queue_create` creates the queue
  - `xe_lrc_create` creates a logical ring context
    - `lrc->bo` holds the ring buffer, sw states (e.g., seqno), etc.
  - `q->ops->init` is usually `guc_exec_queue_init`
    - `xe_sched_init` inits a scheduler with `drm_sched_ops`
      - the wq is `gt->ordered_wq`, an ordered wq with regular priority
    - `xe_sched_entity_init` inits a sched entity

## Jub Submission

- `xe_exec_ioctl`
  - `xe_sched_job_create` allocs a job
  - `xe_sched_job_arm` arms a job
    - `job->lrc_seqno`  is the lrc seqno
    - `job->fence` is a `dma_fence_chain` whose fences are initialized from
      `xe_lrc_init_seqno_fence`
  - `xe_sched_job_push` pushes the job
- `guc_exec_queue_run_job` is called when the job is ready to run
  - `q->ring_ops->emit_job` is `emit_job_gen12_render_compute`
    - `emit_bb_start` emits `MI_BATCH_BUFFER_START`
    - `emit_pipe_imm_ggtt` emits a pipelined write of the lrc seqno
    - `emit_user_interrupt` emits `MI_USER_INTERRUPT`
  - `job->fence` is returned as the hw fence
- on irq,
  - `xe_irq_msix_default_hwe_handler` calls `xe_hw_engine_handle_irq` with
    `GT_RENDER_USER_INTERRUPT`
  - `xe_hw_fence_irq_run` queues `hw_fence_irq_run_cb`
  - `hw_fence_irq_run_cb` signals the fence

## Dumb

- `xe_device_create` has called `ttm_device_init` to init `xe->ttm`
  - `funcs` is `xe_ttm_funcs`
  - `use_dma_alloc` is false
  - `use_dma32` is false
- `xe_device_probe` has called `xe_ttm_sys_mgr_init` to init `xe->mem.sys_mgr`
  - it is used for `XE_PL_TT` aka `TTM_PL_TT`
  - `use_tt` is true
  - `func` is `xe_ttm_sys_mgr_func`
  - `size` is total sysram
- `xe_bo_dumb_create` calls down to `___xe_bo_create_locked` with
  - `bo` is null
  - `tile` is null
  - `resv` is null
  - `bulk` is null
  - `size` is `ALIGN(pitch * height, page)`, where `pitch = ALIGN(width * cpp, 64)`
  - `cpu_caching` is `DRM_XE_GEM_CPU_CACHING_WC`
  - `type` is `ttm_bo_type_device`
  - `flags` is
    - `XE_BO_FLAG_SYSTEM`
    - `XE_BO_FLAG_SCANOUT`
    - `XE_BO_FLAG_NEEDS_CPU_ACCESS`
    - `XE_BO_FLAG_USER`
- `__xe_bo_placement_for_flags` sets `bo->placement` to 1 placement, for
  `XE_BO_FLAG_SYSTEM`
  - `mem_type` is `XE_PL_TT`
  - `flags` is 0
- `ttm_bo_init_reserved` inits bo and holds `bo->resv`
  - the caller must `ttm_bo_unreserve`
  - it calls `ttm_bo_validate`
- `ttm_bo_validate`
  - `ttm_bo_alloc_resource` allocs a res
    - `man->func->alloc` is `xe_ttm_sys_mgr_new`
  - `ttm_bo_handle_move_mem`
    - `ttm_tt_create` creates `bo->ttm`, where `bdev->funcs->ttm_tt_create` is
      `xe_ttm_tt_create`
      - `ttm_tt_init`
        - `page_flags` is `TTM_TT_FLAG_ZERO_ALLOC | TTM_TT_FLAG_EXTERNAL | TTM_TT_FLAG_EXTERNAL_MAPPABLE`
        - `caching` is `ttm_write_combined`
      - `ttm_tt_setup_backup` creates `tt->backup` for swap-out?
    - `ttm_bo_populate` calls `ttm_tt_populate` to populate the backing pages,
      where `bdev->funcs->ttm_tt_populate` is `xe_ttm_tt_populate`
      - it calls `ttm_pool_alloc` to alloc pages from a pool (a page cache)
        - if `global_write_combined` is non-empty, cache hit
        - otherwise, `ttm_pool_alloc_page` calls `alloc_pages_node`
        - `ttm_pool_page_allocated` calls `ttm_pool_allocated_page_commit` to
          add the pages to `alloc->pages` (which points to `tt->pages`)
        - when finally all pages are allocated, `ttm_pool_apply_caching`
    - `bdev->funcs->move` is `xe_bo_move`
      - `xe_tt_map_sg` calls `dma_map_sgtable` to map pages for dma
      - `ttm_bo_move_null` inits `bo->resource`
  - at this point,
    - `bo->ttm` is created and `bo->ttm->pages` is allocated and mapped for dma
    - `bo->resource` is created
      - it seems to "describe" the resource (in this case `bo->ttm`) and is
        quite small
- when the dumb bo is mmaped, `xe_mmap` calls the standard `drm_gem_mmap`
  - `obj->funcs->mmap` is `drm_gem_ttm_mmap`
    - it calls `ttm_bo_mmap_obj` to change vm flags
  - `obj->funcs->vm_ops` is `xe_gem_vm_ops`
    - upon fault, `ttm_bo_vm_fault_reserved` inserts pages
      - `ttm_io_prot` decides the memory type
      - `vmf_insert_pfn_prot` inserts pages
- when the dumb bo is destroyed, `ttm_bo_release` releases the bo
  - `bo->bdev->funcs->release_notify` is `xe_ttm_bo_release_notify`
  - it makes sure the bo is idle (or schedule a work for later instead)
  - `ttm_bo_cleanup_memtype_use`
    - `bo->bdev->funcs->delete_mem_notify` is `xe_ttm_bo_delete_mem_notify`
    - `ttm_bo_tt_destroy`
      - `ttm_tt_unpopulate` calls `xe_ttm_tt_unpopulate`
        - `xe_tt_unmap_sg` unmaps the pages from dma
        - `ttm_pool_free` frees the pages to the pool
      - `ttm_tt_destroy` calls `xe_ttm_tt_destroy`
    - `ttm_resource_free`
      - `man->func->free` is `xe_ttm_sys_mgr_del`
  - `bo->destroy` is `xe_ttm_bo_destroy`
