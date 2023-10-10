Mesa ANV
========

## BO Allocations

- kmd bo allocations are through `anv_kmd_backend::gem_create`
- `anv_device_alloc_bo` is the only wrapper for `gem_create`
  - it returns an `anv_bo` that is refcounted
  - `anv_device_release_bo` frees the bo
  - note that `device->bo_cache.bo_map` is a sparse array of `anv_bo`
    - `anv_device_lookup_bo` maps a gem handle to an `anv_bo`
- direct users of `anv_device_alloc_bo`
  - `anv_CreateDevice` may allocate up to 3 bos for `anv_device`
    - one is used to implement hw workarounds
    - when rt is enable, another two are allocated for ray queries and
      `3DSTATE_BTD` (bindless thread dispatch)
  - `anv_device_init_trivial_batch` allocates a bo for `anv_device`
    - this is used in `setup_empty_execbuf` to submit a valid nop batch
  - `anv_AllocateMemory` allocates a bo for `anv_device_memory`
  - `anv_CreateDescriptorPool` allocates a bo for `anv_descriptor_pool`
  - `genX(CreateQueryPool)` allocates a bo for `anv_query_pool`
  - `anv_image_init` may allocate a bo for `anv_image`
    - this happens only when the image is shared and has an aux surface.  The
      bo is for the aux metadata that is not shared.
  - `anv_bo_sync_init` allocates a bo for `anv_bo_sync`
    - it's used on older i915 kernel driver and relies on bo implicit sync
  - when `INTEL_MEASURE` is set, `anv_measure_init` allocates a bo for
    `anv_cmd_buffer`
  - when rt is enabled,
    - `anv_CmdSetRayTracingPipelineStackSizeKHR` and
      `anv_cmd_buffer_set_ray_query_buffer` may allocate bos for `anv_device`
    - `genX(CmdBuildAccelerationStructuresKHR)` calls
      `anv_cmd_buffer_alloc_space` to allocate two bos for `anv_cmd_buffer`
- indirect users of `anv_device_alloc_bo`
  - `anv_scratch_pool_alloc` allocates a bo for `anv_scratch_pool`
    - it is used for pipeline scratch space
    - there is a scratch pool per `anv_device`
  - `anv_block_pool_expand_range` allocates a bo for `anv_block_pool`
    - a block pool consists of a va range and an array of bos
    - the bos are allocated on demand with `ANV_BO_ALLOC_FIXED_ADDRESS` to be
      the backing storage of the va
    - there is no free
    - it is the base class of `anv_state_pool` and the rest of the driver uses
      `anv_state_pool`
  - `anv_bo_pool_alloc` allocates a bo for `anv_bo_pool`
    - the pool is a bo cache, where freed bos are added to free lists rather
      than freed
    - there are two bo pools per `anv_device`
      - one is for `anv_cmd_buffer` and simple batches
      - one is for utrace ts/cmd buffers
- `anv_state_pool`
  - it is a subclass of `anv_block_pool`
    - a block covers a big range (usually GBs) of fixed va
    - the backing bos are allocated on demand using
      `ANV_BO_ALLOC_FIXED_ADDRESS`
    - unlike `anv_block_pool`, which does not support free, `anv_state_pool`
      supports both alloc and free
  - there are these state pools per `anv_device`
    - ~1GB `general_state_pool`
    - 1GB `low_heap`
    - ~4GB `dynamic_state_pool`
    - 1GB `binding_table_pool`
    - 1GB `internal_surface_state_pool`
    - 8MB `scratch_surface_state_pool`
    - if `indirect_descriptors`
      - 2GB `bindless_surface_state_pool`
      - 3GB `descriptor_pool`
      - 1GB `push_descriptor_pool`
    - else
      - 2GB `descriptor_pool`
    - 2GB `instruction_state_pool`
    - the rest of gtt for `client_visible_heap` and `high_heap`
  - `anv_state_stream` is a wrapper of `anv_state_pool`
    - it functions as a suballocator
    - there is no free and memories are returned in `anv_state_stream_finish`
- `anv_cmd_buffer`
  - there is an `anv_batch`
    - `anv_batch_set_storage` sets the externally-provided storage
    - there is an optional `extend_cb` to allocate the storage
    - `anv_batch_emit_dwords` returns a ptr for writing
  - `anv_cmd_buffer_init_batch_bo_chain` initializes the `anv_batch`
    - it allocates a first `anv_batch_bo`
    - `extend_cb` is set to `anv_cmd_buffer_chain_batch`, which
      - allocates a new `anv_batch_bo`
      - adds the new bbo to `cmd_buffer->seen_bbos` and
        `cmd_buffer->batch_bos`
        - `batch_bos` are bbos owned by the cmdbuf and to be freed with the
          cmdbuf
        - `seen_bbos` includes `batch_bos` as well as bbos from secondary
          cmdbufs, tracked to generate `drm_i915_gem_exec_object2` on submit
      - `cmd_buffer_chain_to_batch_bo` to emit `MI_BATCH_BUFFER_START` to the
        current bbo to jump to the new bbo
      - calls `anv_batch_set_storage` with the new bbo

## Pipelines

- `anv_CreateGraphicsPipelines` calls `anv_graphics_pipeline_create`
- `anv_graphics_pipeline_compile` compiles shaders
  - `anv_graphics_pipeline_load_nir` translates spirv to nir
  - `anv_pipeline_nir_preprocess` preprocesses nir
  - a series of lowering and optimizations
  - `anv_pipeline_compile_vs` and `anv_pipeline_compile_fs` (and others)
    generates the binary
    - they call `brw_compile_vs` and `brw_compile_fs` for codegen, which call
      `brw_postprocess_nir`

## `shaderFloat64`

- newer intel does not support fp64
- some games use fp64 without checking, and `fp64_workaround_enabled` can be
  enabled as a per-game workaround
  - when enabled and when the device does not support fp64, `fp64_nir` is
    created from `src/compiler/glsl/float64.glsl`
  - the glsl file implements various `__fXXX64` functions
  - `nir_lower_doubles_options` has `nir_lower_fp64_full_software`
  - `brw_preprocess_nir` calls `nir_lower_doubles` with `fp64_nir`
- `nir_lower_doubles`
  - `lower_doubles_instr_to_soft` maps `nir_op_XXX` to calls to `__fXXX64` in
    the GLSL

## `shaderInt64`

- newer intel does not support 64-bit integers
  - but anv always advertises `shaderInt64`
  - `brw_compiler_create` enables everything for `nir_lower_int64_options`
  - `brw_preprocess_nir` calls `nir_lower_int64_float_conversions`
  - `brw_postprocess_nir` calls `nir_lower_int64`

## blorp

- `vkCmdCopyBuffer2` flow
  - `blorp_batch_init` initializes `blorp_batch`
  - `blorp_buffer_copy` does the copying
    - `isl_surf_init` to initialize a `isl_surf` on stack for both src/dst
    - `blorp_copy`
      - `blorp_params_init` initializes `blorp_params`
      - `brw_blorp_surface_info_init` initializes `params.src` and
        `params.dst`
      - `blorp_copy_get_formats` overrides formats
      - `blorp_surf_convert_to_uncompressed` overrides w/h for compressed
        formats
      - `genX(blorp_exec)` executes
        - if `BLORP_BATCH_USE_COMPUTE`, `blorp_exec_on_compute` and
          `blorp_exec_compute`
        - else, `blorp_exec_on_render` and `blorp_exec_3d`
  - `blorp_batch_finish` is nop

## Descriptor Sets

- for each descriptor type, `anv_descriptor_data_for_type` decides what does
  the descriptor consist of
  - `ANV_DESCRIPTOR_INDIRECT_SAMPLED_IMAGE` is `anv_sampled_image_descriptor`
  - `ANV_DESCRIPTOR_INDIRECT_STORAGE_IMAGE` is `anv_storage_image_descriptor`
  - `ANV_DESCRIPTOR_INDIRECT_ADDRESS_RANGE` is `anv_address_range_descriptor`
  - because `indirect_descriptors` is usually true, the hw descriptors
    (`RENDER_SURFACE_STATE` and `SAMPLER_STATE`) are not directly written to
    the descriptor sets
- `bindless_surface_state_pool`
  - `anv_physical_device_init_va_ranges` allocates 2GB of `anv_va_range` for
    the pool
  - `anv_state_pool_init` allocates a `anv_state_pool`
    - it's a suballocator that supports alloc/free
    - `start_offset` of the pool is 0, meaning offsets of allocations are
      relative to the pool rather than absolute
  - `anv_state_pool_alloc` allocates from a bucket
    - it tries free list of the bucket first
    - otherwise, it tries the free list of larget buckets, and split the larger
      alloc
    - otherwise, it calls `anv_block_pool_alloc` to allocate from the
      underlying `anv_block_pool`
  - `anv_state_pool_free` always returns to the free list of the bucket
  - `anv_image_view` and `anv_buffer_view` allocate `SURFACE_STATE` from the
    pool
  - `STATE_BASE_ADDRESS::BindlessSurfaceStateBaseAddress` is set to the va of
    the pool
  - `anv_bindless_state_for_binding_table` converts the offset of the surface
    state 
    - from the offset to `bindless_surface_state_pool` to offset to
      `internal_surface_state_pool`
