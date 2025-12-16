Mesa RADV
=========

## Environment Variables

- `RADV_DEBUG` for radv debug flags
  - `RADV_DEBUG=hang` dumps a report after a GPU hang
    - `radv_check_gpu_hangs` is called after each queue submit.  If the submit
      hangs, a report is written to `$HOME/radv_dumps`
  - `RADV_DEBUG=shaders` dumps shader disassembly
    - it calls `nir_print_shader` to dump the final nir before
      `radv_shader_nir_to_asm`
    - if `RADV_DEBUG=preoptir`,
      - it calls `nir_print_shader` again before instruction selection
      - if calls `aco_print_program` after instruction selection
    - it calls `aco_print_program` after RA
    - it calls `get_disasm_string` after codegen
- `RADV_PERFTEST` for radv perf flags
- `ACO_DEBUG` for compiler debug flags
- `MESA_VK_TRACE`
  - see `Tracing` section
- `RADV_FORCE_VRS` and `RADV_FORCE_VRS_CONFIG_FILE` control VRS (variable-rate
  shading)
- `RADV_TRAP_HANDLER` installs a trap handle on GFX8
  - it sets up a trap handler that is invoked when a shader traps
  - after each queue submit, `radv_check_trap_handler` checks if there is any
    fault
- `RADV_TEX_ANISO` forces anisotropy filter
- `RADV_FORCE_FAMILY` enables the null winsys
- `AMD_CU_MASK` can be used to disable CUs
- `AMD_PARSE_IB` can parse the specified binary IB file

## Tracing

- `MESA_VK_TRACE` for a comma-separated list of trace types
  - initializes `instance->trace_mode`
  - common trace types
    - `rmv` for Radeon Memory Visualizer
  - `vk_instance_add_driver_trace_modes` parses driver trace types
    - `rgp` for Radeon GPU Profiler
    - `rra` for Radeon Raytracing Analyzer
    - `ctxroll` for context rolls
- `MESA_VK_TRACE_FRAME` specifies the frame trigger
  - initializes `instance->trace_frame`
- `MESA_VK_TRACE_TRIGGER` specifies the file trigger
  - initializes `instance->trace_trigger_file`
- tracing is triggered from wsi
  - `wsi_common_queue_present` calls `handle_trace`
  - tracing can be triggered for a single frame in a few ways
    - `device->current_frame` is incremented for each frame and triggers
      tracing when `device->current_frame == instance->trace_frame`
    - `instance->trace_trigger_file` is checked for each frame and triggers
      tracing when it exists and is removed
    - `device->trace_hotkey_trigger` is set by a hot key, which is `F1` on X11
  - when triggered, `device->capture_trace` is called
- `RADV_TRACE_MODE_RGP`
  - `radv_amdgpu_winsys_create` is called with a reserved vmid
  - `radv_GetPhysicalDeviceToolProperties` reports the tool
  - `radv_physical_device_get_supported_extensions` disables
    `VK_KHR_performance_query`
  - `init_dispatch_tables` uses the sqtt dispatch table
  - `radv_sqtt_init` initializes sqtt
    - `radv_is_instruction_timing_enabled` is optional and is enabled by
      default
    - `radv_spm_trace_enabled` is optional and is enabled by default
    - `radv_sqtt_queue_events_enabled` is optional and is enabled by default
  - when triggered from wsi, `capture_trace` sets `device->sqtt_triggered`
  - `radv_handle_sqtt` is called after each successful `sqtt_QueuePresentKHR`
    - if sqtt is triggerd for the frame,
      - `radv_begin_sqtt` begins tracing
      - `device->sqtt_enabled` is set
    - if `device->sqtt_enabled` was set for the last frame,
      - `device->sqtt_enabled` is cleared
      - `radv_end_sqtt` ends tracing
      - `radv_get_sqtt_trace` retrives the trace
      - `ac_dump_rgp_capture` dumps the trace
    - that is, tracing is triggered and enabled for a single frame
  - sqtt dispatch table intercepts many calls
    - `sqtt_Create*Pipelines` and `sqtt_DestroyPipeline`
      - `radv_register_pipeline` and `radv_unregister_records`
        register/unregister the pipelines
    - `sqtt_Cmd*` that are actions
      - these mainly records api markers
    - `sqtt_QueuePresentKHR`
      - to be able to call `radv_handle_sqtt` after each present to see if
        tracing needs to be begun/ended
    - `sqtt_QueueSubmit2` records the submit if tracing has begun
- `VK_TRACE_MODE_RMV` is similar
  - `vk_memory_trace_init` initializes the runtime
  - `radv_memory_trace_init` initializes radv-specific stuff
  - when triggered, `radv_rmv_collect_trace_events` collects the events and
    `vk_dump_rmv_capture` dumps them
- `RADV_TRACE_MODE_RRA` is similar
  - `radv_rra_trace_init` initializes rra
  - when triggered, `radv_rra_dump_trace` dumps the accel structs
- `RADV_TRACE_MODE_CTX_ROLLS` is similar

## `libdrm_amdgpu`

- device init
  - `amdgpu_device_initialize` and `amdgpu_device_deinitialize` creates/frees
    an `amdgpu_device`
  - both kernel space and userspace have `struct amdgpu_device`, and
    `amdgpu_device_initialize` makes sure there is a 1:1 mapping within the
    userspace process
    - it uses a global `dev_mutex` to ensure thread-safety
    - it uses `drmGetPrimaryDeviceNameFromFd` to make sure each primary node
      has exactly one `amdgpu_device`
  - both kernel space and userspace have `struct amdgpu_bo`, and
    `libdrm_amdgpu` makes sure there is a 1:1 mapping within the userspace
    process
    - `amdgpu_device` has `bo_handles` and `bo_table_mutex`
    - allocated bos are added to `bo_handles`
    - imported bos are checked before added to `bo_handles`
  - why?
    - when a dma-buf is imported to the same `drm_file` twice, the kernel
      returns the same gem handle
    - if the userspace blindly imports the same dma-buf twice, and then closes
      the same gem handle twice, bad things happen if the gem handle is
      reused for an unrelated bo between the two close calls
    - `bo_handles` and `bo_table_mutex` make sure this cannot happen
  - there is a caveat though
    - if a client opens the same render node twice to get two fds,
      `amdgpu_device_initialize` return the same `amdgpu_device` for the two
      fds
    - `amdgpu_device` does not use the second fd at all and the second
      `drm_file` is ignored completely
- device query
  - `amdgpu_device_get_fd` returns the original fd
  - `amdgpu_get_marketing_name` reutrns the pretty name
  - `amdgpu_query_buffer_size_alignment`
  - `amdgpu_query_firmware_version`
  - `amdgpu_query_gpu_info`
  - `amdgpu_query_heap_info`
  - `amdgpu_query_hw_ip_count`
  - `amdgpu_query_hw_ip_info`
  - `amdgpu_query_info`
  - `amdgpu_query_sensor_info`
  - `amdgpu_query_sw_info`
  - `amdgpu_query_video_caps_info`
  - `amdgpu_read_mm_registers`
  - these are not used by mesa
    - `amdgpu_query_crtc_from_id`
    - `amdgpu_query_gds_info`
    - `amdgpu_query_gpuvm_fault_info`
  - most of them are wrappers to `DRM_AMDGPU_INFO` ioctl
- vm reserve
  - `amdgpu_vm_reserve_vmid` and `amdgpu_vm_unreserve_vmid`
  - they are used by gpu debugger
    - a vmid identifies a paging table
    - normally, each process is randomly assigned a vmid for isolation
    - what this does is to make the randomly assigned vmid the reserved vmid
      of the device
    - all processes that ask for the reserved vmid will thus share the same
      vmid
- va functions
  - `amdgpu_va_range_query` queries offset/size of the regular address space
  - `amdgpu_va_range_alloc` allocates a range from the address spaces
  - `amdgpu_va_range_free` frees a range
  - `amdgpu_va_get_start_addr` returns the start of a range
  - these are entirely userspace
  - there are two address spaces
    - `[virtual_address_offset, virtual_address_max]`
    - `[high_va_offset, high_va_max]`
- BO functions
  - `amdgpu_bo_alloc`
  - `amdgpu_bo_free`
  - `amdgpu_bo_export`
  - `amdgpu_bo_import`
  - `amdgpu_create_bo_from_user_mem`
  - `amdgpu_bo_va_op_raw` manipulates how a bo is mapped in the vm
  - `amdgpu_bo_set_metadata` sets bo metadata
    - some are used by kernel space but most are used by userspace
    - this is used for implicit tiling, which only works with vk dedicated
      alloc
  - `amdgpu_bo_query_info` queries bo info, including metadata
  - these are not used by mesa
    - `amdgpu_bo_inc_ref`
    - `amdgpu_find_bo_by_cpu_mapping`
    - `amdgpu_bo_list_create`
    - `amdgpu_bo_list_create_raw`
    - `amdgpu_bo_list_destroy`
    - `amdgpu_bo_list_destroy_raw`
    - `amdgpu_bo_list_update`
  - these are not used by radv
    - `amdgpu_bo_cpu_map`
    - `amdgpu_bo_cpu_unmap`
    - `amdgpu_bo_wait_for_idle`
    - `amdgpu_bo_va_op`
- CS functions
  - `amdgpu_cs_ctx_create2` creates a hw ctx
    - there is one hw ctx for each priority
  - `amdgpu_cs_ctx_free`
  - `amdgpu_cs_ctx_stable_pstate` changes devfreq
  - `amdgpu_cs_chunk_fence_info_to_data` adds `AMDGPU_CHUNK_ID_IB` to submit
  - `amdgpu_cs_query_fence_status` is used by radv for hang detection
  - `amdgpu_cs_submit_raw2`
  - `amdgpu_cs_create_syncobj2`
  - `amdgpu_cs_destroy_syncobj`
  - `amdgpu_cs_syncobj_export_sync_file`
  - `amdgpu_cs_syncobj_export_sync_file2`
  - `amdgpu_cs_syncobj_import_sync_file`
  - `amdgpu_cs_syncobj_query2`
  - `amdgpu_cs_syncobj_transfer`
  - these are not used by mesa
    - `amdgpu_cs_ctx_create`
    - `amdgpu_cs_ctx_override_priority`
    - `amdgpu_cs_chunk_fence_to_dep`
    - `amdgpu_cs_submit`
    - `amdgpu_cs_submit_raw`
    - `amdgpu_cs_query_reset_state`
    - `amdgpu_cs_wait_fences`
    - `amdgpu_cs_fence_to_handle`
    - `amdgpu_cs_create_semaphore`
    - `amdgpu_cs_destroy_semaphore`
    - `amdgpu_cs_signal_semaphore`
    - `amdgpu_cs_wait_semaphore`
    - `amdgpu_cs_export_syncobj`
    - `amdgpu_cs_syncobj_import_sync_file2`
    - `amdgpu_cs_syncobj_query`
    - `amdgpu_cs_syncobj_reset`
    - `amdgpu_cs_syncobj_signal`
    - `amdgpu_cs_syncobj_timeline_signal`
    - `amdgpu_cs_syncobj_timeline_wait`
  - these are not used by radv
    - `amdgpu_cs_query_reset_state2`
    - `amdgpu_cs_create_syncobj`
    - `amdgpu_cs_import_syncobj`
    - `amdgpu_cs_syncobj_wait`

## Device

- `radv_CreateDevice` calls `init_dispatch_tables` to initialize the dispatch
  tables
  - `device->vk.dispatch_table` is always used
  - when a layer is enabled, its dispatch table is also used
    - its entry points are added to all dispatch tables that are used and
      below it
  - finally, these entry points are added to all used dispatch tables
    - `radv_device_entrypoints`
    - `wsi_device_entrypoints`
    - `vk_common_device_entrypoints`
  - as a result,
    - `device->vk.dispatch_table.Foo` points to the first layer that
      intercepts `vkFoo`
    - let's say the layer is `bar`
    - `device->layer_dispatch.bar.Foo` points to the next layer that
      intercepts `vkFoo`

## Winsys

- `radv_amdgpu_winsys_create` is called for each physical device
  - libdrm's `amdgpu_device_initialize` is called to initialize
    `amdgpu_device_handle`
    - internally, it calls `drmGetPrimaryDeviceNameFromFd` on the fd and make
      sure there is a 1:1 mapping between primary nodes and
      `amdgpu_device_handle`
    - `dev->dev_info` is from kernel's `AMDGPU_INFO_DEV_INFO`
    - `dev->vamgr{,_32,_high,_high_32}` are initialized
      - on a renoir apu,
        - `vamgr_32` is `[2MB, 4GB)`
        - `vamgr` is `[4GB, 128TB)` (48-bit address space)
        - `vamgr_high_32` and `vamgr_high` are similarly sized but the higher
          16 bits are all 1's (that is, about 128TB near the end of 64-bit
          address space)
  - there is a 1:1 mapping between `amdgpu_device_handle` and
    `radv_amdgpu_winsys`
    - `winsys_creation_mutex` and `winsyses` are for that purpose
    - this was done for memory budget
      - <https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/9651>
    - because of this, while two phsyical devices in two vk instances have
      their own `physical_dev->local_fd`, they share the same
      `physical_dev->ws`
      - as such, they are not isolated within the same process
  - `ac_query_gpu_info` is called to initialize `radeon_info`
    - `meminfo` is from kernel's `AMDGPU_INFO_MEMORY`
      - on a renoir apu,
        - `vram.total_heap_size` is 64M (carve-out?)
        - `cpu_accessible_vram.total_heap_size` is also 64M
        - `gtt.total_heap_size` is near 4GB (kernel uses half of usable system
          memory in `amdgpu_ttm_init`)
    - `fix_vram_size` aligns vram size to 256MB
  - `debug_all_bos` is false unless `RADV_DEBUG=allbos`
  - `debug_log_bos` is false unless `RADV_DEBUG=hang`
  - `use_ib_bos` is true unless `RADV_DEBUG=noibs`
- `radv_amdgpu_ctx_create` is called for each vk queue
  - `amdgpu_cs_ctx_create2` makes an `DRM_AMDGPU_CTX` ioctl
  - a fence bo is created
- `radv_amdgpu_winsys_bo_create` is calloed for each alloc
  - `amdgpu_bo_alloc` makes a `DRM_AMDGPU_GEM_CREATE` ioctl
  - `amdgpu_bo_va_op_raw` makes a `DRM_AMDGPU_GEM_VA` ioctl
  - when `debug_log_bos`, `radv_amdgpu_log_bo` logs for bo allocs
  - when `debug_all_bos`, all bos are added to `global_bo_list` automatically
    and their VAs are dumped on gpu hang
- `radv_amdgpu_cs_create` is called for each vk cmd (and internally)
  - `ib_buffer` is the current bo; when it gets full, `radv_amdgpu_cs_grow` is
    called to move it to `old_ib_buffers` and a new bo is allocated
  - `handles` are all bos referenced by the cs; `radv_amdgpu_cs_add_buffer` is
    used to add a bo to `handles`
  - when `use_ib` is true, `radv_amdgpu_cs_grow` emits `PKT3_INDIRECT_BUFFER`
    to jump from the old bo to the new one

## memory types

- gpu mmu
  - gpu uses va
    - the address space is 256TB on `CHIP_RENOIR` and 64GB on `CHIP_STONEY`
    - it is big enough for softpin
  - gpu mmu translates va to pa
    - one range is backed by vram
      - or, with apu, stolen system ram
    - one range is backed on pcie gart
  - vram can be physical or carved out
    - if carved out, the size is configurable, but can be as small as 16MB or
      64MB
    - if physical, only a portion is accessible by cpu through pci bar
  - pcie gart
    - seems to be 512MB or 1024MB
  - gtt is the larger of `AMDGPU_DEFAULT_GTT_SIZE_MB` (3GB) and `sysram/2`
    - it is the size of system ram that can be mapped into GART
- radv sums visible VRAM (0.5GB) and GTT (3GB) and sets up two fake heaps
  - first is non-local `RADV_HEAP_GTT` heap that gets 1.166GB
  - second is local `RADV_HEAP_VRAM_VIS` heap that gets 2.333GB
  - if `radv_enable_unified_heap_on_apu` is set, there is no gtt heap
- it also sets up 8 memory types
  - 1st is `RADEON_DOMAIN_VRAM` and `RADEON_FLAG_NO_CPU_ACCESS` to
    `RADV_HEAP_VRAM_VIS`
  - 2nd is `RADEON_DOMAIN_GTT`, `RADEON_FLAG_GTT_WC`, and
    `RADEON_FLAG_CPU_ACCESS` to `RADV_HEAP_GTT`
  - 3rd is `RADEON_DOMAIN_VRAM` and `RADEON_FLAG_CPU_ACCESS` to
    `RADV_HEAP_VRAM_VIS`
  - 4th is `RADEON_DOMAIN_GTT` and `RADEON_FLAG_CPU_ACCESS` to
    `RADV_HEAP_GTT`
- `radv_get_memory_budget_properties`
  - `RADEON_ALLOCATED_VRAM`, `RADEON_ALLOCATED_VRAM_VIS`, and
    `RADEON_ALLOCATED_GTT` are tracked by winsys and are the sums of
    - `RADEON_DOMAIN_VRAM + RADEON_FLAG_NO_CPU_ACCESS` bos,
    - `RADEON_DOMAIN_VRAM + !RADEON_FLAG_NO_CPU_ACCESS` bos, and
    - `RADEON_DOMAIN_GTT` bos respectively
  - `RADEON_VRAM_USAGE`, `RADEON_VRAM_VIS_USAGE`, and `RADEON_GTT_USAGE` are
    tracked by kernel and are `AMDGPU_INFO_VRAM_GTT` plus
    - `AMDGPU_INFO_VRAM_USAGE`,
    - `AMDGPU_INFO_VIS_VRAM_USAGE`, and
    - `AMDGPU_INFO_GTT_USAGE` respectively

## Heaps and Memory Mapping

- radv heaps and memory types are
  - heap 0: vram minus cpu-accessible-vram
  - heap 1: cpu-accessible-vram
  - heap 2: gtt (`radeon_info` calls it gart, but it is gtt)
  - type 0 is called VRAM: use heap 0
  - type 1 is called `GTT_WRITE_COMBINE`: use heap 2 w/ coherency w/o cache
  - type 2 is called `VRAM_CPU_ACCESS`: use heap 1 w/ coherency w/o cache
  - type 3 is called `GTT_CACHED`: use heap 2 w/ coherency w/ cache
- `vkAllocateMemory`
  - userspace is responsbile for managing the VM of GPU for itself
  - it find a VA in the VM
  - it allocate a BO with `DRM_IOCTL_AMDGPU_GEM_CREATE`
  - it maps the BO into the VM with `DRM_IOCTL_AMDGPU_GEM_VA`
- External Memory Import
  - getting the BO handle
    - for prime, it is the standard `DRM_IOCTL_PRIME_FD_TO_HANDLE`
    - for userptr, it is `DRM_IOCTL_AMDGPU_GEM_USERPTR`
  - it is then followed by finding a VA for the imported BO and map it into
    the VM
- `vkMapMemory`
  - if userptr, return directly
  - otherse, `DRM_IOCTL_AMDGPU_GEM_MMAP` followed by an mmap

## Queues

- `radv_get_physical_device_queue_family_properties` enumerates up to 5 queue
  families
  - family 0 is `RADV_QUEUE_GENERAL`
  - family 1 is `RADV_QUEUE_COMPUTE`, if supported and not disabled
  - family 2 is `RADV_QUEUE_VIDEO_DEC`, if supported and enabled
  - family 3 is `RADV_QUEUE_TRANSFER`, if supported and enabled
  - family 4 is `RADV_QUEUE_SPARSE`
    - unless `radv_legacy_sparse_binding` is set, this is the only queue
      supporting `VK_QUEUE_SPARSE_BINDING_BIT`
- `radv_queue_init` picks `radv_queue_submit` or `radv_queue_sparse_submit`
  depending on the queue family
  - the device timeline and submit modes are
    `VK_DEVICE_TIMELINE_MODE_ASSISTED` and
    `VK_QUEUE_SUBMIT_MODE_THREADED_ON_DEMAND`
    - or, on older kernels, `VK_DEVICE_TIMELINE_MODE_EMULATED` and
      `VK_QUEUE_SUBMIT_MODE_DEFERRED`
  - `vk_queue_init` converts `VK_QUEUE_SUBMIT_MODE_THREADED_ON_DEMAND` to
    `VK_QUEUE_SUBMIT_MODE_IMMEDIATE`
    - `vk_queue_submit` will call `vk_queue_enable_submit_thread` on demand to
      convert to `VK_QUEUE_SUBMIT_MODE_THREADED`
    - `vk_queue_enable_submit_thread` can also be explicitly called
- `radv_queue_submit`
  - if legacy sparse binding is enabled,
    `radv_queue_submit_bind_sparse_memory` updates sparse bindings
  - `radv_update_preambles` initializes preamble cs
    - this initializes per-queue `initial_full_flush_preamble_cs`,
      `initial_preamble_cs` and `continue_preamble_cs`
  - cs are chained and submitted to winsys, together with the wait/signal
    semaphores
- `radv_queue_sparse_submit`
  - `radv_queue_submit_bind_sparse_memory` updates sparse bindings
  - this uses `DRM_AMDGPU_GEM_VA` ioctl with `AMDGPU_VA_OP_REPLACE`
    - it uses implicit fencing to make sure the va is replaced after the bo is
      idle
    - there is ongoing work to switch to explicit fencing

## Formats

- `radv_physical_device_get_format_properties`
  - if unrecognized, (1-plane) subsampled, or etc2, bail immediately
  - if planar, xfers, sampling, disjoint are supported
    - and early return
  - if `radv_is_storage_image_format_supported`, storage is supported
  - if `radv_is_vertex_buffer_format_supported`, vb is supported
  - if `radv_is_buffer_format_supported`, tbo is supported
  - if ds
    - if `radv_is_zs_format_supported`, ds, sampling, blit, xfer, are supported
    - if d+s, only blit src is supported
  - if color
    - if `radv_is_sampler_format_supported`, sampling and blit src (except for
      `R32G32B32`) are supported
    - if `radv_is_colorbuffer_format_supported` and not 64-bit channel, rb and
      blit dst are supported
    - if supported and is not sscale/uscale, xfer is supported
  - if `radv_is_atomic_format_supported`, atomic is supported
- `radv_is_vertex_buffer_format_supported`
  - it supports 8/16/32/64-bit channels and more
  - `radv_write_vertex_descriptors` builds the descriptor
- `radv_is_sampler_format_supported`
  - `radv_translate_tex_dataformat` returns
    `V_008F14_IMG_DATA_FORMAT_8_8_8_8`, etc.
  - `radv_translate_tex_numformat` returns `V_008F14_IMG_NUM_FORMAT_UNORM`,
    etc.
    - unorm/snorm/float/srgb supports linear filtering
    - uint/sint supports nearest
    - uscaled/sscaled are not supported
  - `radv_make_texture_descriptor` builds the descriptor
  - aco `visit_tex` uses `aco_opcode::image_load*`,
    `aco_opcode::image_sample*`, or `aco_opcode::image_gather4*`
- `radv_is_storage_image_format_supported`
  - this looks like a subset of `radv_is_sampler_format_supported`
    - unorm/snorm/float are supported
    - uint/sint are supported
    - srgb/uscaled/sscaled are not supported
    - compressed are not supported
  - `radv_make_texture_descriptor` builds the descriptor
  - aco `visit_image_load` and `visit_image_store` use
    `aco_opcode::image_load*` and `aco_opcode::image_store*`
- `radv_is_atomic_format_supported`
  - this is limited to 32-bit int/float and 64-bit int
- `radv_is_filter_minmax_format_supported`
  - this is limited to required formats
  - `radv_init_sampler` builds the descriptor
- `radv_is_buffer_format_supported`
  - `radv_translate_buffer_dataformat` returns
    `V_008F0C_BUF_DATA_FORMAT_8_8_8_8`, etc.
  - `radv_translate_buffer_numformat` returns `V_008F0C_BUF_NUM_FORMAT_UNORM`,
    etc.
  - `radv_make_texel_buffer_descriptor` builds the descriptor
  - aco `visit_tex` uses `aco_opcode::buffer_load_format*`
    - for ssbo, aco uses `aco_opcode::buffer_store_*` and
      `aco_opcode::buffer_load_*`
- `radv_is_colorbuffer_format_supported`
  - `ac_get_cb_format` returns `V_028C70_COLOR_8_8_8_8`, etc.
    - channel sizes
  - `radv_translate_colorswap` returns `V_028C70_SWAP_STD`, etc.
    - channel order
  - `ac_get_cb_number_type` returns `V_028C70_NUMBER_UNORM`, etc.
    - numeric format
  - `radv_initialise_color_surface` builds `CB_COLOR_INFO`
- `radv_is_zs_format_supported`
  - `radv_translate_dbformat` returns `V_028040_Z_16`, etc.
  - `radv_initialise_ds_surface` builds `DB_Z_INFO` and `DB_STENCIL_INFO`

## Image Creation

- a `radv_image` has an array of `radv_image_plane`
- each `radv_image_plane` has a `radeon_surf`
  - radv fills in `flags` and `modifier`
  - `radv_amdgpu_winsys_surface_init` calls `ac_compute_surface` to initialize
    the rest
- `radeon_surf::flags`
  - bit 0..7: type
    - e.g., `RADEON_SURF_TYPE_2D`
  - bit 8..15: mode
    - `RADEON_SURF_MODE_LINEAR_ALIGNED` if linear
    - `RADEON_SURF_MODE_2D` if tiled
  - bit 16: `RADEON_SURF_SCANOUT` if scanout
  - bit 17: `RADEON_SURF_ZBUFFER` if depth
  - bit 18: `RADEON_SURF_SBUFFER` if stencil
  - bit 21: `RADEON_SURF_FMASK` is unused
  - bit 22: `RADEON_SURF_DISABLE_DCC` disables DCC compression
    - see `radv_use_dcc_for_image_early` and `radv_use_dcc_for_image_late` for
      reasons to enable/disable DCC
  - bit 23: `RADEON_SURF_TC_COMPATIBLE_HTILE` uses TC-compatible HTILE (for
    sampling)
  - bit 24: `RADEON_SURF_IMPORTED` is imported, unused by radv
  - bit 25: `RADEON_SURF_CONTIGUOUS_DCC_LAYERS` is always set
  - bit 26: `RADEON_SURF_SHAREABLE` is shareable, unused by radv
  - bit 27: `RADEON_SURF_NO_RENDER_TARGET` disables render target
  - bit 28: `RADEON_SURF_FORCE_SWIZZLE_MODE` forces swizzle mode, unused by
    radv
  - bit 29: `RADEON_SURF_NO_FMASK` disables FMASK
    - see `radv_use_fmask_for_image` for reasons to enable/disable FMASK
  - bit 30: `RADEON_SURF_NO_HTILE` disables HTILE (hiz)
  - bit 31: `RADEON_SURF_FORCE_MICRO_TILE_MODE` forces micro tile mode, unused
    by radv
  - bit 32: `RADEON_SURF_PRT` is an layout for sparse images
- `ac_compute_surface`
  - radv is resposible to initialize `ac_surf_info` and these fields in
    `radeon_surf`
    - `flags`
    - `modifier`
    - `blk_w`, block width
    - `blk_h`, block height
    - `bpe`, block size in bytes
  - a micro tile mode specifies the preferred swizzle
    - `RADEON_MICRO_MODE_DISPLAY` prefers `ADDR_SW_*_D`
    - `RADEON_MICRO_MODE_STANDARD` prefers `ADDR_SW_*_S`
    - `RADEON_MICRO_MODE_DEPTH` prefers `ADDR_SW_*_Z`
    - `RADEON_MICRO_MODE_RENDER` prefers `ADDR_SW_*_R`
- `radv_GetImageSubresourceLayout`
  - GFX9+
    - `ac_surface_get_plane_offset` for offset
    - `surface->u.gfx9.surf_pitch` or `surface->u.gfx9.pitch` for pitch
  - GFX8-
    - `surface->u.legacy.level[level].offset` for offset
    - `surface->u.legacy.level[level].nblk_x` for pitch
- RETILE
  - when RETILE is false, DCC may or may not be aligned
  - when RETILE is true, there are two DCC surfaces
    - one is unaligned (but displayable)
    - one is aligned
- depth/stencil
  - `RADEON_SURF_ZBUFFER` is set when there is depth
  - `RADEON_SURF_SBUFFER` is set when there is stencil
  - `gfx9_compute_surface`
    - `gfx9_compute_miptree` is called for depth and stencil separately
      - called twice if both surfaces exist
      - only one of `AddrSurfInfoIn.flags.depth` and
        `AddrSurfInfoIn.flags.stencil` is set in each call
  - `gfx6_compute_surface`
    - `gfx6_compute_level` is called for depth and stencil separately
      - this is similiar to gfx9+
      - only one of `AddrSurfInfoIn.flags.depth` and
        `AddrSurfInfoIn.flags.stencil` is set in each call
    - there are two special flags for depth+stencil before gfx9
      - before gfx9, DB uses the same pitch and the same tile mode for
        depth+stencil
      - we need two flags to make depth+stencil TC-compatible, where depth and
        stencil use separate descriptors
        - `noStencil` disables (when set) or enables (when unset) depth pitch
          padding
          - without the padding, DB uses the unpadded pitch for both depth and
            stencil and causes the stencil to be TC-incompatible
          - with the padding, DB uses the padded pitch for both depth and
            stencil and causes the depth to be TC-incompatible if mipmapped
        - `matchStencilTileCfg` potentially downgrades the depth tile mode
          such that it matches the stencil's tile mode
          - this is known to be required only when the depth is mipmapped or
            when `CHIP_STONEY`
      - when depth only, we want to set `noStencil` and unset
        `matchStencilTileCfg`
        - `noStencil` avoids unnecessary depth pitch padding that wastes
          memory
        - no `matchStencilTileCfg` avoids picking a suboptimal tile mode for
          the depth
      - when depth+stencil without mipmapping, we want to unset `noStencil`
        and unset `matchStencilTileCfg`
        - no `noStencil` pads the depth pitch to match stencil's
          - there is no mipmapping so TC is still happy
        - no `matchStencilTileCfg` picks an optimal depth tile mode, which is
          known to match stencil's when not mipmapping
        - except on `CHIP_STONEY`, we still unset `noStencil` but we set
          `matchStencilTileCfg`
          - `CHIP_STONEY` requires `matchStencilTileCfg` to pick matching
            tile mode
      - when depth+stencil with mipmapping, we want to set `noStencil`
        and set `matchStencilTileCfg`
        - with both set, we make the depth TC-compatible and the stencil
          potentially TC-incompatible
        - if the stencil is indeed TC-incompatible, `stencil_adjusted` is set
      - for example, on `CHIP_STONEY`, a plain 8x16 `VK_FORMAT_D16_UNORM_S8_UINT` has
        - `ADDR_COMPUTE_SURFACE_INFO_INPUT`
          - `tileMode` is `ADDR_TM_2D_TILED_THIN1` (that is, macro tiled)
          - `tileType` is `ADDR_DEPTH_SAMPLE_ORDER` (that is, Z tiled)
          - `matchStencilTileCfg` is set
          - `noStencil` is set
            - This is a bug.  It should be set iff mipmapping.
        - `OptimizeTileMode` forces `ADDR_TM_1D_TILED_THIN1` (micro tiled)
          because the surface is too small
        - `HwlGetSizeAdjustmentMicroTiled`
          - on gfx8, this function padds pitch such that slice size is aligned
            to `baseAlign` which is 256 (`m_pipeInterleaveBytes`)
          - in our case, the depth is `8*16*2=256` bytes and the stencil will
            be `8*16*1=128` bytes
          - if `noStencil` is unset, because stencil pitch will be padded to
            16, the depth pitch is also padded to 16
            - note that pitch is in pixels and `baseAlign` is in bytes
        - IOW,
          - if `noStencil` is set, depth pitch is 8 and stencil pitch is 16
            - DB `S_028058_PITCH_TILE_MAX` will be 8, and DB assumes stencil
              is also 8
            - stencil texture descriptor `S_008F20_PITCH` will be 16 and
              stencil sampling will break
          - if `noStencil` is unset, depth pitch is 16 and stencil pitch is 16
            - DB `S_028058_PITCH_TILE_MAX` will be 16
            - depth sampling will break if mipmapping
- `gfx6_compute_surface`
  - `tileMode` is determined first
  - `tileType` is determined next
  - `gfx6_compute_level` is called on each level
    - for the main surface, `AddrComputeSurfaceInfo` is called
      - `surf->surf_size` is increased for each level
      - slices of a level are packed together
    - if dcc, `AddrComputeDccInfo` is called
      - `surf->meta_size` is increased for each level if `subLvlCompressible`
    - if htile, `AddrComputeHtileInfo` is called
      - `surf->meta_size` is set
  - if stencil, `gfx6_compute_level` is called on each level
    - if d+s, stencil is completey separated from depth
  - if fmask, `AddrComputeFmaskInfo` is called
    - `surf->fmask_size` is set
  - if dcc and mipmapped, `surf->meta_size` is overridden
    - it looks like each 1 byte in dcc covers a 256-byte microtile
  - if htile and mipmapped, `surf->meta_size` is overridden
    - it looks like each dword in htile covers a 8x8 tile
  - if cmask, `ac_compute_cmask` is called
    - `surf->cmask_size` is set
- `gfx9_compute_surface`
  - `swizzleMode` is determined
  - `gfx9_compute_miptree` is called
    - for the main surface, `Addr2ComputeSurfaceInfo` is called
      - `surf->surf_size` is set
    - if htile, `Addr2ComputeHtileInfo` is called
      - `surf->meta_size` is set
    - if dcc, `Addr2ComputeDccInfo` is called
      - `surf->meta_size` is set
    - if fmask, `Addr2ComputeFmaskInfo` is called
      - `surf->fmask_size` is set
    - if cmask, `Addr2ComputeCmaskInfo` is called
      - `surf->cmask_size` is set
  - if stencil, `gfx9_compute_miptree` is called again
    - if d+s, stencil is completey separated from depth
- DCC retile
  - the render engine prefers the DCC blocks to be pipe (L2) aligned and rb
    aligned
  - the display engine might require the DCC blocks to be packed (neither pipe
    aligned nor rb aligned)
  - in `ac_query_gpu_info`
    - `use_display_dcc_with_retile_blit` is set when there is enough CU
      - when set, there are both aligned and unaligned DCCs
        - the driver renders to the aligned DCC first and blits to unaligned DCC
    - `use_display_dcc_unaligned` is set when there is 1 RB and is GFX9
      - when set, use unaligned dcc if scanout
      - when cleared, disable dcc if scanout
      - `use_display_dcc_with_retile_blit` takes precedence
    - IOW, when an image is for scanout,
      - if `use_display_dcc_with_retile_blit`, there are an aligned and an
        unaligned DCC
      - if `use_display_dcc_unaligned`, there is an unaligned DCC
      - else, there is no DCC

## Image Layout

- the goal of layout transition is to
  - resolve DCC/FMASK/CMASK for color buffers
    - DCC is for compression
    - FMASK is for fast clear and msaa
    - CMASK is for msaa
  - resolve HTILE for depth buffers
    - HTILE is for hiz
- from `VK_IMAGE_LAYOUT_UNDEFINED`
  - `radv_init_color_image_metadata` initializes DCC/FMASK/CMASK
  - `radv_initialize_htile` initializes HTILE
- `radv_layout_is_htile_compressed` determines which layouts can use HTILE
  - most layouts can use HTILE as long as HTILE is TC-compatible, as
    returned by `radv_image_is_tc_compat_htile`
  - but if HTILE is not tc-compat, only ds optimal layouts and xfer dst can
    use HTILE (I guess it uses 3d pipeline for xfer dst)
- `radv_expand_depth_stencil` resolves HTILE when transitioning away from
  HTILE
  - on `RADV_QUEUE_GENERAL`, `radv_process_depth_stencil` resolves using the
    3d pipeline
    - `radv_get_depth_pipeline` returns a pipeline
      - vs is passthrough
      - fs is nop
      - `DB_RENDER_CONTROL` has special bits
        - `depth_compress_disable` sets `DEPTH_COMPRESS_DISABLE`
        - `stencil_compress_disable` sets `STENCIL_COMPRESS_DISABLE`
    - draw a rectlist
    - the aspect mask is ignored even when expanding depth and stencil
      separately
      - should we change `depth_compress_disable` / `stencil_compress_disable`?
  - otherwise, `radv_expand_depth_stencil_compute` resolves using the compute
    pipeline
    - it sets up the image for load and store
    - load respects htile
      - this requires a tc-compat htile
    - store ignores htile
- `radv_layout_dcc_compressed` determines which layouts can use DCC
  - most layouts use DCC as long as the image supports DCC
- `radv_decompress_dcc` resolves DCC when transitioning away from DCC
  - on `RADV_QUEUE_GENERAL`, `radv_process_color_image` resolves using the
    3d pipeline
  - otherwise, `radv_decompress_dcc_compute` resolves using the compute
    pipeline
- `radv_layout_can_fast_clear` determines which layouts can use fast clear
  - only optimal attachment layouts can use fast clear
- `radv_fast_clear_flush_image_inplace` resolves fast clear when transitioning
  away from fast clear
  - `radv_fast_clear_eliminate` resolves CMASK
  - `radv_fmask_decompress` resolves FMASK
- `radv_layout_fmask_compressed`
- `radv_expand_fmask_image_inplace`

## External Images

- `radv_GetPhysicalDeviceFormatProperties2` calls
  `radv_list_drm_format_modifiers` to query modifiers
  - it rejects compressed or depth/stencil formats
  - `ac_get_supported_modifiers` returns all supported modifiers
    - on gfx9,
      - `AMD_FMT_MOD_TILE_GFX9_64K_D_X` with and without dcc
      - `AMD_FMT_MOD_TILE_GFX9_64K_S_X` with and without dcc
      - if block size is 32 bits, `AMD_FMT_MOD_TILE_GFX9_64K_S_X` with dcc
        retile
      - `AMD_FMT_MOD_TILE_GFX9_64K_D` without dcc
      - `AMD_FMT_MOD_TILE_GFX9_64K_S` without dcc
      - `DRM_FORMAT_MOD_LINEAR`
  - `radv_get_modifier_flags` removes `DISJOINT` support
  - if planar, `drmFormatModifierPlaneCount` is set to format plane count
    - there is no dcc support (but the function does not reject them atm)
  - if non-planar, `drmFormatModifierPlaneCount` is
    - 3 if dcc retile
    - 2 if dcc
    - 1 if no dcc
- `radv_GetPhysicalDeviceImageFormatProperties2`
  - `radv_get_image_format_properties` handles most properties and call
    `radv_check_modifier_support` for modifiers
    - it accepts only 2D, non-sparse
    - if dcc, `radv_are_formats_dcc_compatible` checks
      `VK_IMAGE_CREATE_MUTABLE_FORMAT_BIT`
    - if dcc, non-array and non-mipmap
    - no msaa
  - `get_external_image_format_properties` handles external properties
    - if dma-buf, it requires
      - `VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT`
      - 2D
    - if opaque fd and non-linear, it also requires dedicated alloc
- amdgpu supports per-bo `amdgpu_bo_metadata` for use by userspace
  - it is still in use, until the entire userspace becomes explicit
    - iow, it defines `DRM_FORMAT_MOD_INVALID`
  - on radeonsi export, `si_texture_get_handle` calls `si_set_tex_bo_metadata`
    - `ac_surface_compute_umd_metadata` essentially embeds the texture
      descriptor in the opaque metadata
    - `ac_surface_compute_bo_metadata` computes `tiling_info` from
      `radeon_surf`
    - when the resource is multi-planar,
      - `whandle->plane` specifies the plane
      - `resource` points to the specified plane
      - `si_set_tex_bo_metadata` is called only if this is the first plane
        - `whandle->offset == 0`
  - on radeonsi import, `si_texture_from_winsys_buffer` calls
    `buffer_get_metadata`
    - `ac_surface_apply_bo_metadata` partially inits `radeon_surf` from
      `tiling_info`
    - `ac_surface_apply_umd_metadata` partially inits `radeon_surf` from the
      opaque metadata
- if a memory is dedicated,
  - `radv_GetMemoryFdKHR` calls `buffer_set_metadata`
    - `radv_init_metadata` initializes the metadata
      - the function itself computes the 64-bit tiling info, that both kernel
        and userspace understand
        - unlike radeonsi, it does not use `ac_surface_compute_bo_metadata`
      - `radv_query_opaque_metadata` computes the 256-byte opaque info,
        that only userspace understands
    - `radv_amdgpu_winsys_bo_set_metadata` saves the metadata in the bo
  - `radv_AllocateMemory` with `VkImportMemoryFdInfoKHR` calls
    `buffer_get_metadata` if not `VK_IMAGE_TILING_DRM_FORMAT_MODIFIER_EXT`
    - `radv_amdgpu_winsys_bo_get_metadata` decodes `tiling_info`
      - unlike radeonsi, it does not use `ac_surface_apply_bo_metadata`
    - `radv_image_create_layout` resets the layout and calls
      `ac_surface_apply_umd_metadata` to apply the opaque metadata
- comparing legacy metadta and modifiers
  - `AMDGPU_TILING_SWIZZLE_MODE` becomes `AMD_FMT_MOD_TILE`
  - `AMDGPU_TILING_DCC_OFFSET_256B` becomes `AMD_FMT_MOD_DCC` and a memory
    plane
  - `AMDGPU_TILING_DCC_PITCH_MAX` becomes a memory plane
  - `AMDGPU_TILING_DCC_INDEPENDENT_64B` becomes
    `AMD_FMT_MOD_DCC_INDEPENDENT_64B`
  - `AMDGPU_TILING_DCC_INDEPENDENT_128B` becomes
    `AMD_FMT_MOD_DCC_INDEPENDENT_128B`
  - `AMDGPU_TILING_SCANOUT` has no replacement
    - a wsi image is allocated with the scanout bit
    - an external image won't have the scanout bit
      - it is the allocator's responsibility to pick a modifier that supports
        scanout
  - `ac_surface_apply_umd_metadata` handles opaque metadata
    - `G_008F1C_LAST_LEVEL` has no replacement
    - `G_008F1C_TYPE` has no replacement
      - `radv_check_modifier_support` limits to non-MSAA
    - `G_008F28_COMPRESSION_EN` becomes `AMD_FMT_MOD_DCC`
    - `G_008F24_META_DATA_ADDRESS` becomes a memory plane
    - `G_008F24_META_PIPE_ALIGNED` becomes `AMD_FMT_MOD_DCC_PIPE_ALIGN`
    - `G_008F24_META_RB_ALIGNED` becomes `AMD_FMT_MOD_DCC_PIPE_ALIGN`

## Command Processor

- `radeon_set_config_reg`
  - GFX6-only
  - reg range is `[SI_CONFIG_REG_OFFSET, SI_CONFIG_REG_END]`, 12KB
  - op is `PKT3_SET_CONFIG_REG`
- `radeon_set_context_reg`
  - reg range is `[SI_CONTEXT_REG_OFFSET, SI_CONTEXT_REG_END]`, 32KB
  - op is `PKT3_SET_CONTEXT_REG`
- `radeon_set_sh_reg`
  - reg range is `[SI_SH_REG_OFFSET, SI_SH_REG_END]`, 4KB
  - op is `PKT3_SET_SH_REG` or `PKT3_SET_SH_REG_INDEX` (since GFX10)
- `radeon_set_uconfig_reg`
  - reg range is `[CIK_UCONFIG_REG_OFFSET, CIK_UCONFIG_REG_END]`, 64KB
  - op is `PKT3_SET_UCONFIG_REG` or `PKT3_SET_UCONFIG_REG_INDEX` (since newer
    GFX9)

## Command Buffer

- most `radv_CmdSetFoo` saves the state and marks the dirty bits
  - e.g., `radv_CmdSetDepthBiasEnable` saves the bool and marks
    `RADV_CMD_DIRTY_DYNAMIC_DEPTH_BIAS_ENABLE`
- from `radv_before_draw`, `radv_emit_all_graphics_states` emits all dirty
  states
  - e.g., if `RADV_CMD_DIRTY_DYNAMIC_DEPTH_BIAS_ENABLE` (or some other dirty
    bits) is set, `radv_emit_culling` sets `R_028814_PA_SU_SC_MODE_CNTL` reg

## Timestamps

- `timestampValidBits` is 64
- `timestampPeriod` is `1000000 / clock_crystal_freq`
  - `clock_crystal_freq` is from `drm_amdgpu_info_device::gpu_counter_freq` in
    khz
  - 100mhz on renoir, 25mhz on raven, 48mhz on stoney
- timestamp is queried with `DRM_AMDGPU_INFO(AMDGPU_INFO_TIMESTAMP)`
  - `gfx_v8_0_get_gpu_clock_counter`
    - writes `mmRLC_CAPTURE_GPU_CLOCK_COUNT`
    - reads `mmRLC_GPU_CLOCK_COUNT_LSB` and `mmRLC_GPU_CLOCK_COUNT_MSB`
  - `gfx_v9_0_get_gpu_clock_counter`
    - if `GC_HWIP` is 9.3.0,
      - reads `mmGOLDEN_TSC_COUNT_LOWER_Renoir` and
        `mmGOLDEN_TSC_COUNT_UPPER_Renoir`
    - if `GC_HWIP` is 9.0.1,
      - writes `mmRLC_CAPTURE_GPU_CLOCK_COUNT`
      - reads `mmRLC_GPU_CLOCK_COUNT_LSB` and `mmRLC_GPU_CLOCK_COUNT_MSB`

## AHB

- export
  - `radv_android_gralloc_supports_format` (and others) are used to limit the
    support
  - `vk_image_init` initializes `ahardware_buffer_format` and
    `android_external_format` to 0
  - `radv_ahb_format_for_vk_format` is used set `ahardware_buffer_format`
    - in this path, the formats are limited by
      `radv_android_gralloc_supports_format`
  - `radv_create_ahb_memory` is used to allocate the memory
    - what it does is to allocate an ahb and import
  - export returns the allocated ahb
- import
  - `vk_format_from_android` is used to get the vk format
    - external format is also the vk format
    - the vk format may not have a matching public ahb format
- allocate externally and import
  - `vk_image_usage_to_ahb_usage` is used to initialize
    `VkAndroidHardwareBufferUsageANDROID`

## MSAA and Sample Shading

- `VkPipelineMultisampleStateCreateInfo` is translated to
  `vk_multisample_state`
  - `rasterizationSamples` to `rasterization_samples`
    - sets `key.ps.num_samples` which affects the ps interp instruction
    - `radv_emit_default_sample_locations` emits the default sample locs
    - `radv_emit_msaa_state` emits
      - `R_028804_DB_EQAA`
      - `R_028BE0_PA_SC_AA_CONFIG`
      - `R_028A48_PA_SC_MODE_CNTL_0`
  - `sampleShadingEnable` to `sample_shading_enable`
    - sets `key.ps.sample_shading_enable` which affects
      `nir_intrinsic_load_sample_mask_in`
    - `radv_get_ps_iter_samples` returns a `ps_iter_samples` higher than 1
    - `radv_emit_rasterization_samples` emits
      - `R_0286E0_SPI_BARYC_CNTL`
      - `R_028A4C_PA_SC_MODE_CNTL_1`
  - `minSampleShading` to `min_sample_shading`
      - it uses `rasterization_samples` and `color_samples` to determine the
        sample count
      - then `util_next_power_of_two(ceilf(sample_count * min_sample_shading))`
  - `pSampleMask` to `sample_mask`
    - `radv_emit_sample_mask` emits the sample mask
  - `alphaToCoverageEnable` to `alpha_to_coverage_enable`
    - sets `ps_epilog.need_src_alpha` for mrt0
    - sets `key.ps.alpha_to_coverage_via_mrtz` on gfx11+
  - `alphaToOneEnable` to `alpha_to_one_enable`
    - radv does not support `alphaToOne` feature

## Sampling

- when a shader samples a simple image...
- `image_sample` is the instruction used
  - the operands are
    - dst, to specify the destination
    - coords, to specify the coords
    - resource, to specify the image
    - sampler, to specify the sampler
  - stack
    - `aco::(anonymous namespace)::emit_mimg`
    - `aco::(anonymous namespace)::visit_tex`
    - `aco::(anonymous namespace)::visit_block`
    - `aco::(anonymous namespace)::visit_cf_list`
    - `aco::select_program`
    - `aco_compile_shader`
    - `shader_compile`
    - `radv_shader_nir_to_asm`
- there is an image descriptor
  - `radv_image_view_make_descriptor` initializes the image descriptor
    - `radv_make_texture_descriptor`
    - `si_set_mutable_tex_desc_fields`
- there is a sampler descriptor
  - `radv_init_sampler` initializes the sampler descriptor

## Compute Pipelines

- `radv_CreateComputePipelines` calls `radv_compute_pipeline_create` for each
  pipeline
  - `radv_pipeline_init` initializes the struct
  - `radv_compute_pipeline_compile` compiles the shader
    - `radv_pipeline_stage_init` initializes a local `radv_shader_stage`
    - `radv_compile_cs` compiles
  - `radv_compute_pipeline_init` emits the pm4 packets
- `radv_compile_cs`
  - `radv_shader_spirv_to_nir` translates spirv to nir, lowers nir, and calls
    `radv_optimize_nir` to apply common optimizations
  - `nir_shader_gather_info` reflects the nir shader and initializes the
    generic `nir->info`
  - `radv_nir_shader_info_pass` reflects the nir shader and init radv-specific
    `radv_shader_info`
  - `radv_declare_shader_args` uses the reflections to initalize
    `radv_shader_args`
  - `radv_postprocess_nir` lowers nir again before sending it to the backend
    compiler
  - `radv_shader_nir_to_asm` compiles nir to `radv_shader_binary`
    - `shader_compile` calls either `aco_compile_shader` or
      `llvm_compile_shader`
    - `radv_postprocess_binary_config` updates `ac_shader_config` associated
      with the binary
  - `radv_shader_create` creates a `radv_shader` to wrap `radv_shader_binary`
    and upload the binary to a bo
- `radv_nir_shader_info_pass`
  - `info->cs.block_size` is initialized from `nir->info.workgroup_size`
  - `gather_shader_info_cs` initializes `info->wave_size`
    - it uses `VkPipelineShaderStageRequiredSubgroupSizeCreateInfo` if
      specified,
    - it forces to `RADV_SUBGROUP_SIZE` (64) if
      `VK_PIPELINE_SHADER_STAGE_CREATE_REQUIRE_FULL_SUBGROUPS_BIT` is set or
      if the shader uses features that require full subgroup
    - it uses 32 if supported and the local workgroup size is small, or
    - it uses `pdev->cs_wave_size` which is 64 unless `RADV_PERFTEST=cswave32`
      is set
- `radv_shader_create`
  - `radv_get_max_waves` computes the max waves
    - this does not compute the wave size (64 or 32), but the max number of
      waves per simd32 which determines occupancy
    - `gpu_info->max_waves_per_simd` is 16 on rdna2+ and 20 on rdna
    - the use of lds affects max waves
    - the use of sgprs affects max waves before rdna
      - on rdna2+, there are 16x128 sgprs and a shader can use at most 128
        sgprs.  It does not affect the result.
    - the use of vgprs affects max waves
      - on rdna2+, there are 1024 vgprs and a shader can use up to 512(?)
        vgprs
    - `gpu_info->num_simd_per_compute_unit` is 2
      - `simd_per_workgroup` is doubled on rdna because CU count reported by
        amdgpu is actually WGP count, and each WGP consists of a pair of CUs
  - `radv_alloc_shader_memory` suballocates a bo
  - `radv_shader_upload` memcpys the binary to the bo

## Graphics Pipelines

- `radv_CreateGraphicsPipelines` calls `radv_graphics_pipeline_create` for
  each pipeline
- `radv_graphics_pipeline_init` initializes a pipeline
  - `radv_pipeline_import_graphics_info` parses `VkGraphicsPipelineCreateInfo`
  - `radv_generate_graphics_pipeline_key` generates `radv_pipeline_key`
  - `radv_graphics_pipeline_compile` compiles the pipeline
  - `radv_pipeline_emit_pm4` emits the pm4 packets
- `radv_graphics_pipeline_compile`
  - `radv_pipeline_get_nir` calls `radv_shader_spirv_to_nir` to convert spirv
    to nir
  - `radv_graphics_pipeline_link` calls `radv_pipeline_link_shaders` on all
    shader paris to link their ios
  - `radv_fill_shader_info` calls `radv_nir_shader_info_pass` and
    `radv_nir_shader_info_link` to initialize `radv_shader_info`
  - `radv_declare_pipeline_args`
  - `radv_postprocess_nir` post-processes each stage
  - `radv_pipeline_nir_to_asm` calls `radv_shader_nir_to_asm` to convert nir
    to asm
- `radv_shader_nir_to_asm`
  - `radv_fill_nir_compiler_options` initializes `radv_nir_compiler_options`
  - `radv_aco_convert_opts` converts `radv_nir_compiler_options` to
    `aco_compiler_options`
  - `radv_aco_convert_shader_info` converts `radv_shader_info` to
    `aco_shader_info`
  - `aco_compile_shader` (or `llvm_compile_shader`) compiles nir to asm
  - `radv_postprocess_binary_config` updates `binary->config`
    - for aco, `binary->config` is kept updated
    - for llvm, `ac_rtld_read_config` initializes `binary->config` from elf
      `.AMDGPU.config` section
- `aco_compile_shader`
  - `aco::select_program` translates nir to aco ir
  - `aco_postprocess_shader` schedules aco ir, performs reg alloc, optimizes,
    eliminates pseudo ops, etc.
  - `aco::emit_program` emits the binary
  - `radv_aco_build_shader_binary` allocs a `radv_shader_binary_legacy` to
    hold the binary
  - see <radv-aco.md>
- prolog and epilog
  - I guess
    - vs can get inputs from fixed-function or from a "prolog" shader
      - the inputs are known as
        `VK_GRAPHICS_PIPELINE_LIBRARY_VERTEX_INPUT_INTERFACE_BIT_EXT`
    - rb can get outputs from fixed-function or from a "epilog" shader
      - the outputs are known as
        `VK_GRAPHICS_PIPELINE_LIBRARY_FRAGMENT_OUTPUT_INTERFACE_BIT_EXT`
    - `radv_generate_graphics_pipeline_key` determines `vs.has_prolog` and
      `ps.has_epilog`
  - `key.vs.has_prolog` means the pipeline needs a vs epilog
    - it is true when VI is dynamic (`RADV_DYNAMIC_VERTEX_INPUT`)
    - `radv_emit_vertex_input` will create and emit the vs prolog at draw time
  - `key.ps.has_epilog` means the pipeline needs a ps epilog
    - it is true when color output states are dynamic
      (`radv_pipeline_needs_dynamic_ps_epilog`)
    - `radv_emit_all_graphics_states` emits the ps epilog at draw time
      - when `key.ps.dynamic_ps_epilog` is cleared, the pipeline comes with a
        ps epilog created by `radv_pipeline_create_ps_epilog`
      - otherwise, the ps epilog is created at draw time
- at the end of pipeline compilation, `radv_emit_vertex_shader` emits the pm4
  packets
  - `as_es` is set when the consumer is gs
    - it emits vs as "ES"
  - `as_ls` is set when the consumer is tcs
    - it emits vs as "LS"
  - `use_ngg` is set on gfx10+
    - ngg stands for Next-Gen Geometry
    - it is introduced in gfx10 and is the only supported mechanism in gfx11

## Pipeline Statistics

- `radv_GetPipelineExecutableStatisticsKHR` returns pipeline stats that are
  useful for shader-db
- `Driver pipeline hash` is the pipeline hash reported to RGP
- `SGPRs` is the number of sgprs used
- `VGPRs` is the number of vgprs used
- `Spilled SGPRs` is the number of sgprs spilled
- `Spilled VGPRs` is the number of vgprs spilled
- `Code size` is the binary size in bytes
- `LDS size` is the lds size used
- `Scratch size` is the scratch size used
- `Subgroups per SIMD` is the max occupancy determined by `radv_get_max_waves`
- ACO-specific
  - `collect_presched_stats` is called before spill and scheduling
    - `aco_statistic_sgpr_presched` is `Pre-Sched SGPRs`, number of sgprs
      before scheduling
    - `aco_statistic_vgpr_presched` is `Pre-Sched VGPRs`, number of vgprs
      before scheduling
  - `lower_to_hw_instr`
    - `aco_statistic_copies` is `Copies`
  - `collect_preasm_stats` is called before codegen
    - `aco_statistic_instructions` is `Instructions`, number of instrs
    - `aco_statistic_branches` is `Branches`, number of branches
    - `aco_statistic_valu` is `VALU`, number of valu instrs
    - `aco_statistic_salu` is `SALU`, number of salu instrs
    - `aco_statistic_vopd` is `VOPD`, number of vopd instrs
    - `aco_statistic_vmem_clauses` is `VMEM Clause`
    - `aco_statistic_vmem` is `VMEM`, number of vmem instrs
    - `aco_statistic_smem_clauses` is `SMEM Clause`
    - `aco_statistic_smem` is `SMEM`, number of smem instrs
    - `aco_statistic_latency` is `Latency`, estimated cycles per invocation
    - `aco_statistic_inv_throughput` is `Inverse Throughput`, estimated cycles
      per wave64
  - `collect_postasm_stats` is called after codegen
    - `aco_statistic_hash` is `Hash`

## Vertex Prolog

- `VK_DYNAMIC_STATE_VERTEX_INPUT_EXT`
  makes `VkPipelineVertexInputStateCreateInfo` fully dynamic
  - it is translated to `RADV_DYNAMIC_VERTEX_INPUT` internally
  - it sets `key.vs.has_prolog`
    - `gather_shader_info_vs` will set `info->vs.has_prolog` and
      `info->vs.dynamic_inputs`
  - `radv_CmdSetVertexInputEXT` sets up VI dynamically
    - `state->dynamic_vs_input` is initialized
    - `RADV_CMD_DIRTY_DYNAMIC_VERTEX_INPUT` is set
  - at draw time, `radv_emit_vertex_input` emits vi state
    - it is nop when no prolog is needed
- `radv_nir_lower_vs_inputs` calls lowers `lower_load_vs_input` or
  `lower_load_vs_input_from_prolog` depending on whether there is a vs prolog

## Clears

- vk has 3 ways to clear
  - `radv_CmdBeginRendering` calls `radv_cmd_buffer_clear_rendering` to clear
    attachments
    - it calls `radv_subpass_clear_attachment` for each attachment which calls
      `emit_clear`
  - `radv_CmdClearAttachments` calls `emit_clear` to clear an attachment
  - `radv_CmdClearColorImage` and `radv_CmdClearDepthStencilImage` call
    `radv_cmd_clear_image` to clear an image
    - it calls `radv_clear_image_layer` which calls `emit_clear`
- `emit_clear` is called by all 3 methods
  - `radv_fast_clear_color` or `emit_color_clear` for color buffers
  - `radv_fast_clear_depth` or `emit_depthstencil_clear` for zs
- `radv_can_fast_clear_color`
  - the image view must support fast clear
    - `radv_image_view_can_fast_clear` requires the image to support fast
      clear and the view covers the entire image
    - `radv_image_can_fast_clear` requires
      - dcc or cmask for colors
      - htile for zs
  - the image layout must support fast clear
    - `radv_layout_can_fast_clear` for colors
    - `radv_layout_is_htile_compressed` for zs
  - the clear must cover the whole image
  - the clear value must be supported
  - more
- `radv_can_fast_clear_depth` is similar
  - the image view must support fast clear
  - the image layout must support fast clear
  - the clear must cover the whole image
  - the clear value must be supported
    - `radv_is_fast_clear_depth_allowed` requires 1.0 or 0.0
    - `radv_is_fast_clear_stencil_allowed` requires 0
- `radv_fast_clear_depth` fast clears a depth attachment
  - `radv_get_htile_fast_clear_value` returns the htile value
    - `zmask` is 0, meaning the depth is fast-cleared
    - `smem` is 0, meaning the stencil is fast-cleared
  - `radv_clear_htile` clears the htile buffer
  - `radv_update_ds_clear_metadata` updates the clear value
    - `radv_set_ds_clear_metadata` writes the clear value to the image
      metadata
      - there are 2 dwords per miplevel of metadata following the image data
      - `radv_get_ds_clear_value_va` returns the va
    - `radv_update_tc_compat_zrange_metadata` writes a special value to the
      image metadata
      - the special value is 0 or `UINT32_MAX`, and is for a hw bug workaround
      - `radv_get_tc_compat_zrange_va` returns the va
    - `radv_update_bound_fast_clear_ds` writes the clear value to the
      `DB_STENCIL_CLEAR` and `DB_DEPTH_CLEAR` registers
      - `radv_update_zrange_precision` applies the hw bug workaround
  - `radv_load_ds_clear_metadata` is called at draw time
    - it loads the fast clear value from the image metadata to the registers
- `emit_depthstencil_clear` slow clears a depth attachment
  - it draws a rect using an internal pipeline
  - `create_depthstencil_pipeline` creates the pipeline
    - `use_rectlist` is always set, to draw a rect
    - `db_depth_clear` and `db_stencil_clear` are set if the depth attachment
      supports fast clear
      - they translate to `DEPTH_CLEAR_ENABLE` and `STENCIL_CLEAR_ENABLE`
      - I guess when those bits are set, the clear value from the
        `DB_STENCIL_CLEAR` and `DB_DEPTH_CLEAR` registers rather than from the
        fs

## Performance Counters

- when a physical device is created, `ac_init_perfcounters` is called to
  initialize hw counters
- counter enumeration returns an array of pseudo counters
  - all pseudo counters are
    - `VK_PERFORMANCE_COUNTER_SCOPE_COMMAND_KHR`
    - `VK_PERFORMANCE_COUNTER_STORAGE_FLOAT64_KHR`
    - `VK_PERFORMANCE_COUNTER_DESCRIPTION_CONCURRENTLY_IMPACTED_BIT_KHR`
  - `radv_init_perfcounter_descs` initializes the pseudo counters
    - the value of a pseudo counter is calculated from a list of pseudo regs
    - `radv_perfcounter_op` defines the calculations (sum, max, etc.)
    - a pseudo reg is a 32-bit value
      - bit 31: constant
        - if set, the lower 31 bits are a constant uint
        - if clear, see below
      - bit 16..30: block
        - `enum ac_pc_gpu_block`
      - bit0..15: selector (defined by hw)
- multi-pass queries
  - when a query reads X pseudo counters, it is translated to read Y pseudo
    regs
  - if a pseudo reg is non-constant, it belongs to some `ac_pc_gpu_block`
  - there is a limit on how many hw counters can be active at a time for each
    gpu block
  - if a query requires more hw counters than a gpu block has, the query is
    multi-pass
  - `radv_get_counter_registers` translates pseudo counters to pseudo regs
  - `radv_get_num_counter_passes` calculates the pass count
- when the perf query feature is enabled,
  - `device->perf_counter_bo` is allocated
  - `device->perf_counter_lock_cs` is allocated
- when a query pool for `VK_QUERY_TYPE_PERFORMANCE_QUERY_KHR` is created, a
  `radv_pc_query_pool` rather than a `radv_query_pool` is created
  - `radv_pc_init_query_pool` initializes the query pool
    - `radv_get_counter_registers` translates pseudo counters to pseudo regs
    - `radv_get_num_counter_passes` inits pass count
    - pseudo regs are translated to hw regs
    - a hw reg can have N instances
      - when there are X units in Y SEs, `N = X * Y`
    - `pool->stride` is the size of 1 query
- profiling lock uses `AMDGPU_CTX_OP_SET_STABLE_PSTATE` to set
  `AMDGPU_CTX_STABLE_PSTATE_PEAK`
- begin/end query
  - `radv_pc_begin_query`
    - it writes 0 to offset `PERF_CTR_BO_FENCE_OFFSET` of `device->perf_counter_bo`
    - it sets `R_036020_CP_PERFMON_CNTL`, `R_037390_RLC_PERFMON_CLK_CNTL`,
      `R_031100_SPI_CONFIG_CNTL`, and `R_036780_SQ_PERFCOUNTER_CTRL`
    - it emits `PKT3_COND_EXEC` based on offset `PERF_CTR_BO_PASS_OFFSET` of
      `device->perf_counter_bo` to select different counters
    - `radv_pc_stop_and_sample` takes a snapshot
  - `radv_pc_end_query`
    - it writes 1 to offset `PERF_CTR_BO_FENCE_OFFSET` of `device->perf_counter_bo` and waits
    - `radv_pc_stop_and_sample` takes another snapshot
- submit
  - `radv_create_perf_counter_lock_cs` creates two shorts cs to be executed
    before and after
    - they manipulate offset `PERF_CTR_BO_LOCK_OFFSET` and
      `PERF_CTR_BO_PASS_OFFSET` for multi-pass queries
- getting result
  - `radv_pc_get_results` performs the calculations

## NGG & GDS

- `radv_cmd_buffer::gds_needed`
  - it is set when streamout or when NGG with certain query types are enabled
  - when queue submit, `radv_update_preamble_cs` allocates `queue->gds_bo`
- gs pipeline stats
  - `radv_create_query_pool` sets `pool->uses_gds` when gs
    primitives/invocations are enabled on gfx10.3
  - `emit_begin_query` calls `gfx10_copy_gds_query` to override pipeline stats
    - `V_028A90_SAMPLE_PIPELINESTAT` writes the stats to the pool
    - `gfx10_copy_gds_query` overrides gs invocations/primitives
  - `emit_end_query` is similar
  - `radv_nir_lower_abi` handles two related intrinsics
    - `nir_intrinsic_atomic_add_gs_emit_prim_count_amd` increments
      `RADV_SHADER_QUERY_GS_PRIM_EMIT_OFFSET` in gds
    - `nir_intrinsic_atomic_add_shader_invocation_count_amd` increments
      `RADV_SHADER_QUERY_GS_INVOCATION_OFFSET` in gds
  - those two intrinsics are emitted by `ac_nir_gs_shader_query`

## LLVM

- radv integraion with ac llvm
  - it calls `ac_init_llvm_once` on-demand to initialize LLVM
  - it calls `llvm_compile_shader` to compile nir to binary
    - it calls `radv_init_llvm_compiler` to initialize a per-thread compiler
      on-demand
      - it calls `ac_init_llvm_compiler` to inialize a `ac_llvm_compiler`
      - it calls `ac_create_llvm_passes` to create a `ac_compiler_passes`
    - it calls `ac_translate_nir_to_llvm` to translate `nir_shader` to
      `LLVMModuleRef`
    - it calls `radv_compile_to_elf` to compile llvm ir to binary
      - it calls `ac_compile_module_to_elf` to do the work
- IOW, these functions from ac llvm are used
  - `ac_init_llvm_once` calls `ac_init_llvm_target` once
    - when a process loads both the gpu and the video drivers, and when shared
      llvm is enabled, it has a hack to make sure `ac_init_llvm_target` is
      still called once
      - it makes `ac_init_shared_llvm_once` public, such that both drivers
        resolve to the same definition of `ac_init_shared_llvm_once`
    - `ac_llvm_run_atexit_for_destructors` makes sure llvm's static variables
      are destructed after mesa's atexit handlers
  - `ac_init_llvm_compiler` calls
    - `ac_create_target_machine` to create a `LLVMTargetMachineRef`
    - `ac_create_target_library_info` to create a `LLVMTargetLibraryInfoRef`
    - `ac_create_passmgr` to create a `LLVMPassManagerRef`
      - this passmgr is used after nir to llvm ir translation
  - `ac_create_llvm_passes` creates a `legacy::PassManager`
    - this passmgr is used to codegen
  - various functions from `ac_llvm_build.c`
    - they translate nir to llvm ir
  - `ac_compile_module_to_elf` compiles llvm ir to binary

## `radv_CmdCopyImage2`

- `copy_image` executes each `VkImageCopy2` one by one
  - each `VkImageCopy2` may copy multiple layers/slices
  - a 2d image can have multiple layers
  - a 3d image can have multiple slices, but always has 1 layer
- `blit_surf_for_image_level_layer` returns a `radv_meta_blit2d_surf` to
  describe a layer/slice
- `radv_meta_blit2d_rect` is in texel blocks
- there is a loop to copy each layer/slice one by one
  - `radv_meta_blit2d_surf::layer` is incremented each time

## Barriers

- `radv_cs_emit_cache_flush` emits flush commands to the cs
  - L1
    - `RADV_CMD_FLAG_INV_ICACHE` invalidates I$
    - `RADV_CMD_FLAG_INV_SCACHE` invalidates scalar L1
    - `RADV_CMD_FLAG_INV_VCACHE` invalidates vector L1
  - L2, one of
    - `RADV_CMD_FLAG_INV_L2` flushes and invalidates L2
    - `RADV_CMD_FLAG_WB_L2` flushes L2
    - `RADV_CMD_FLAG_INV_L2_METADATA` invalidates image metadata
  - CB/DB
    - `RADV_CMD_FLAG_FLUSH_AND_INV_CB` flushes and invalidates CB cache
    - `RADV_CMD_FLAG_FLUSH_AND_INV_DB` flushes and invalidates DB cache
  - barrier
    - `RADV_CMD_FLAG_PS_PARTIAL_FLUSH` waits for PS idle
    - `RADV_CMD_FLAG_VS_PARTIAL_FLUSH` waits for VS idle
    - `RADV_CMD_FLAG_CS_PARTIAL_FLUSH` waits for CS idle
    - `RADV_CMD_FLAG_VGT_FLUSH` waits for VGT (vertex/geometry translator)
      idle
  - pipeline stats
    - `RADV_CMD_FLAG_START_PIPELINE_STATS`
    - `RADV_CMD_FLAG_STOP_PIPELINE_STATS`
- `radv_CmdPipelineBarrier2` (and other commands) sets flush bits in
  `cmd_buffer->state.flush_bits`
  - it also handles image transitions, but that's entirely separated
- before draw or dispatch, `radv_emit_cache_flush` calls
  `radv_cs_emit_cache_flush` to flush `cmd_buffer->state.flush_bits`

## fp16

- glsl `f16vec2`
  - `a * b` is translated to nir `16x2  %3 = fmul %1, %2`
    - `nir_lower_alu_width` does not scalarize it because
      - `vectorize_vec2_16bit` returns 2 for the fmul
      - `opt_vectorize_callback` also returns 2 for the fmul
    - aco selects `aco_opcode::v_pk_mul_f16` for the instruction
  - `f16vec2(vec2(a, b))` is translated to nir `16x2  %2 = f2f16 %1`
    - `nir_lower_alu_width` scalarizes it to `con 16  %3 = f2f16 %2.x` and
      `con 16  %4 = f2f16 %2.y` because, while `vectorize_vec2_16bit` returns
      2, `opt_vectorize_callback` returns 1 because
      `aco_nir_op_supports_packed_math_16bit` returns false
    - aco selects `aco_opcode::v_cvt_f16_f32` for the two f2f16
      - fwiw, the hw only supports `aco_opcode::v_cvt_pkrtz_f16_f32` which
        forces rtz
  - when there is a phi node for the f16vec2,
    - `nir_lower_phis_to_scalar` scalarizes the phi node
      - this seems to cause inefficiency

## Registers

- `parse_kernel_headers.py` generates json files from kernel headers
- `parse_kernel_headers.py gfx103 ~/projects/linux/drivers/gpu/drm/amd/include > gfx103.json`
  - `asic_reg/gc/gc_10_3_0_offset.h`
    - reg `mmFOO` has two defines, `mmFOO` and `mmFOO_BASE_IDX`
    - the real offset is `bases[mmFOO_BASE_IDX] + mmFOO`
    - the bases are hardcoded
      - in kernel, they are parsed from in-memory `ip_discovery.bin`
    - `register_filter` filters out most registers
  - `asic_reg/gc/gc_10_3_0_sh_mask.h`
    - reg `mmFOO` has multiple fields, with each field having
      `FOO__BAR__SHIFT` and `FOO__BAR_MASK`
  - `navi10_enum.h` has a lot of enum types
    - `enum_map` maps fields to enum types
