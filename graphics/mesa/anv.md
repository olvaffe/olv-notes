Mesa ANV
========

## Initialization

- `anv_physical_device::info` is a `struct intel_device_info`
  - it is initialized by `intel_get_device_info_from_fd` which calls
    `intel_device_info_init_common` to map pci id to one of
    `intel_device_info_*`

## Memory Heaps and Types

- `anv_physical_device_init_heaps` initializes the memory heaps and types
  - it calls KMD-specific funtions
  - on x86, `SUPPORT_INTEL_INTEGRATED_GPUS` is set; host-visible but
    non-coherent mt is allowed
    - it sets `device->memory.need_flush` and ends up issuing (potentially
      multiple) `mfence` and `clflush` for cpu cache flush/invalidate
- `anv_i915_physical_device_init_memory_types`
  - if `anv_physical_device_has_vram`, there are 3 memory types
    - device local
    - host visible+coherent+cached
    - device local and host visible+coherent
    - the condition is true for discrete gpus
  - if `device->info.has_llc`, there are also 3 memory types
    - device local
    - device local and host visible+coherent
    - device local and host visible+coherent+cached
    - the condition is true before xe and non-atom
  - otherwise, there are 2 memory types
    - device local and host visible+coherent
    - device local and host visible+cached
  - if `device->has_protected_contexts`, there is one extra memory type
    - device local+protected
    - the condition is true on gfx12 with newer i915
- `anv_xe_physical_device_init_memory_types`

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

## BO Flags

- `ANV_BO_ALLOC_32BIT_ADDRESS` uses `vma_lo` heap and clears
  `EXEC_OBJECT_SUPPORTS_48B_ADDRESS`
- `ANV_BO_ALLOC_EXTERNAL` must be set for bos that are shared
  - it affects host cacheability/coherency if allocated by the app
    - ignore the requested memory type
    - always `ANV_BO_ALLOC_HOST_COHERENT`
    - no `ANV_BO_ALLOC_HOST_CACHED`
  - it affects mocs
    - use WT
  - it affects pat entry
    - treated as SCANOUT
  - it affects mmap mode
    - use WC
- `ANV_BO_ALLOC_MAPPED` makes sure the bo is mappable and mapped
- `ANV_BO_ALLOC_HOST_COHERENT` makes sure the bo is coherent
  - no effect to the kmd
  - affects pat entry
- `ANV_BO_ALLOC_CAPTURE` means `EXEC_OBJECT_CAPTURE`, to include the bo in
  devcoredump
- `ANV_BO_ALLOC_FIXED_ADDRESS` uses the requested bo addr
- `ANV_BO_ALLOC_IMPLICIT_SYNC` clears `EXEC_OBJECT_ASYNC`
  - execbuffer will honor the implicit fences associated with the bo
- `ANV_BO_ALLOC_IMPLICIT_WRITE` means `EXEC_OBJECT_WRITE`
  - execbuffer will add an implicit write fence rather than read fence
- `ANV_BO_ALLOC_CLIENT_VISIBLE_ADDRESS` picks a bo address that is unlikely to
  have a conflict
- `ANV_BO_ALLOC_AUX_TT_ALIGNED` aligns to AUX-TT, which is always set for app
  allocs
- `ANV_BO_ALLOC_LOCAL_MEM_CPU_VISIBLE` picks visible vram
- `ANV_BO_ALLOC_NO_LOCAL_MEM` never picks vram
- `ANV_BO_ALLOC_SCANOUT`
- `ANV_BO_ALLOC_DESCRIPTOR_POOL` picks `vma_desc` for descriptor pools
- `ANV_BO_ALLOC_TRTT` picks `vma_trtt` for sparse
  - i915 only supports TRTT
  - xe supports `vm_bind` (default) and TRTT
- `ANV_BO_ALLOC_PROTECTED` means `I915_GEM_CREATE_EXT_PROTECTED_CONTENT`
- `ANV_BO_ALLOC_HOST_CACHED` uses a cached host mapping
- `ANV_BO_ALLOC_SAMPLER_POOL` picks `vma_samplers` for samplers
- `ANV_BO_ALLOC_IMPORTED`
  - it affects pat entry
    - always cached/coherent
  - it affects mmap mode
- `ANV_BO_ALLOC_INTERNAL` has no real effect but only for rmv debug tool

## BO Addresses

- `anv_device_alloc_bo`, rather than i915, picks the bo address
  - `bo->offset` is initialized by `anv_bo_vma_alloc_or_close` to be the
    canonical address of the bo
    - on i915, this makes use of `EXEC_OBJECT_PINNED`
  - `anv_vma_heap_for_flags` picks the heap
    - for regular `VkDeviceMemory`, `device->vma_hi` is used
    - for `VkDescriptorPool`, `device->vma_desc` is used
    - for `VK_MEMORY_ALLOCATE_DEVICE_ADDRESS_BIT`, `device->vma_cva` is used
      - this is used in trace capture
  - there is also `explicit_address` support
    - in trace replay, `VkMemoryOpaqueCaptureAddressAllocateInfo` specifies an
      explicit address in `device->vma_cva` heap to use
    - `anv_block_pool` (base class of `anv_state_pool`), which does not
      support free, uses `ANV_BO_ALLOC_FIXED_ADDRESS` to skip vma heap
- `struct anv_address` is `anv_bo` plus an `offset`
  - `anv_address_physical` returns `bo->offset + offset` as the 64-bit address
  - it is necessary because i915 requires all active bos to be tracked so we
    want to express an address as a bo plus an offset
- `anv_state_pool_alloc` and `anv_state_stream_alloc` return an `struct anv_state`
  - it includes
    - `offset` is relative to the `anv_state_pool`
    - `alloc_size` is the size of the alloc and can be 0 to indicate
      `ANV_STATE_NULL`
    - `map` is the cpu pointer
    - `idx` is for internal use (for free)
  - `anv_state_pool_state_address` returns `anv_address` of `anv_state`

## Compute Pipelines

- `anv_CreateComputePipelines` calls `anv_compute_pipeline_create`
  - `anv_pipeline_init` performs the common init
  - `anv_pipeline_init_layout` inits `pipeline->layout` from
    `anv_pipeline_layout`
  - `anv_pipeline_compile_cs`
    - `anv_pipeline_stage_get_nir` translated spirv to nir
    - `anv_pipeline_nir_preprocess` preprocesses nir
    - `anv_pipeline_lower_nir` lowers nir
    - `brw_compile_cs` generates the binary
    - `anv_device_upload_kernel` uploads the binary to
      `instruction_state_pool`
    - `anv_pipeline_add_executables` saves stats for
      `VK_KHR_pipeline_executable_properties`
  - `genX(compute_pipeline_emit)` emits `MEDIA_VFE_STATE` and
    `INTERFACE_DESCRIPTOR_DATA` to `pipeline->batch`
- `anv_pipeline_lower_nir` and resources
  - `anv_nir_compute_used_push_descriptors` returns a bitmask of used
    descriptors in the push descriptor set
  - `anv_nir_apply_pipeline_layout`
  - `anv_nir_update_resource_intel_block`
  - `anv_nir_compute_push_layout`
  - `anv_nir_lower_resource_intel`
  - `anv_nir_loads_push_desc_buffer`
  - `anv_nir_push_desc_ubo_fully_promoted`

## Graphics Pipelines

- `anv_CreateGraphicsPipelines` calls `anv_graphics_pipeline_create`
- `anv_graphics_pipeline_compile` compiles shaders
  - `anv_graphics_pipeline_load_nir` translates spirv to nir
  - `anv_pipeline_nir_preprocess` preprocesses nir
  - a series of lowering and optimizations
  - `anv_pipeline_compile_vs` and `anv_pipeline_compile_fs` (and others)
    generates the binary
    - they call `brw_compile_vs` and `brw_compile_fs` for codegen, which call
      `brw_postprocess_nir`
  - `anv_device_upload_kernel` uploads the binary to `instruction_state_pool`

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

- `anv_CreateDescriptorPool`
  - `anv_descriptor_data_for_type` returns the kinds of data each descriptor
    includes
    - `ANV_DESCRIPTOR_INDIRECT_SAMPLED_IMAGE` is `anv_sampled_image_descriptor`
    - `ANV_DESCRIPTOR_INDIRECT_STORAGE_IMAGE` is `anv_storage_image_descriptor`
    - `ANV_DESCRIPTOR_INDIRECT_ADDRESS_RANGE` is `anv_address_range_descriptor`
    - because `indirect_descriptors` is usually true, the hw descriptors
      (`RENDER_SURFACE_STATE` and `SAMPLER_STATE`) are not included in the
      descriptors
  - `anv_descriptor_data_size` returns the hw size of a descriptor
  - `pool->host_heap` and `pool->host_mem_size`
    - the pool has `host_mem_size` of cpu memory that follows `pool`
      immediately
    - `host_heap` suballocates from the cpu memory
    - the cpu memory is used for
      - `maxSets` of `struct anv_descriptor_set`
      - one `struct anv_descriptor` for each descriptor
      - one `struct anv_buffer_view` for each descriptor that includes
        `ANV_DESCRIPTOR_BUFFER_VIEW` (that is, ubo and ssbo)
  - `pool->bo`, `pool->bo_heap`, and `pool->bo_mem_size`
    - `bo` has size `bo_mem_size` and is allocated with
      `ANV_BO_ALLOC_DESCRIPTOR_POOL` to use `vma_desc` heap
    - `bo_heap` suballocates from `bo`
  - `pool->surface_state_stream` uses `device->internal_surface_state_pool`
- `anv_AllocateDescriptorSets`
  - `anv_descriptor_set_layout_size` returns the required host mem size for
    the layout
    - the `struct anv_descriptor_set` itself
    - one `struct anv_descriptor` for each descriptor
    - one `struct anv_buffer_view` for each descriptor that includes
      `ANV_DESCRIPTOR_BUFFER_VIEW` (that is, ubo and ssbo)
  - `anv_descriptor_pool_alloc_set` allocates a `struct anv_descriptor_set`
    from `host_heap`
    - `set->descriptors` follows `set`
    - `set->buffer_views` follows `set->descriptors`
  - `anv_descriptor_set_layout_descriptor_buffer_size` returns the required hw
    mem size for the layout
  - `set->desc_mem` is suballocated from `set->bo` using `set->bo_heap`
    - each suballocation is aligned to `ANV_UBO_ALIGNMENT`
    - `set->desc_mem` and `set->desc_addr` are manually initialized
    - `set->desc_offset` is relative to `internal_surface_state_pool`
- `STATE_BASE_ADDRESS::SurfaceStateBaseAddress` before gfx 12.5
  - binding tables must be in the first 64KB of `SurfaceStateBaseAddress`
    - because `3DSTATE_BINDING_TABLE_POINTERS_*` only has 16 bits for the
      binding table offsets which are relative to `SurfaceStateBaseAddress`
  - each entry of the binding table is a 32-bit offset relative to
    `SurfaceStateBaseAddress`
  - `binding_table_pool` is special
    - `base_address` is at the top (that is,
      `internal_surface_state_pool.adddr`
    - `start_offset` is negative (`base_address + start_offset` is
      `binding_table_pool.addr`)
    - `block_size` is `BINDING_TABLE_POOL_BLOCK_SIZE` (64KB)
  - `flush_descriptor_sets`
    - if there is no space for binding tables,
      - `anv_cmd_buffer_new_binding_table_block` allocates
        `BINDING_TABLE_POOL_BLOCK_SIZE` (64KB) from `binding_table_pool`
      - `genX(cmd_buffer_emit_state_base_address)` updates
        `SurfaceStateBaseAddress` to `anv_cmd_buffer_surface_base_address`
        which is the the new 64KB block
    - `anv_cmd_buffer_alloc_binding_table` allocates a binding table from the
      64KB block and returns the offset from the 64KB block to
      `internal_surface_state_pool`
    - `emit_binding_table` updates each entry of the binding table
      - `surface_state` is relative to `internal_surface_state_pool`
      - that's why it does `state_offset + surface_state.offset` to get an
        offset relative to `SurfaceStateBaseAddress`
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

## DRM modifiers

- `anv_GetPhysicalDeviceFormatProperties2`
  - it calls `anv_get_image_format_features2` with different tilings/modifiers
    to query their features
    - `isl_drm_modifier_info_for_each` loops through all known modifiers
    - it does not always treat `DRM_FORMAT_MOD_LINEAR` as linear tiling, so
      there might be bugs
    - 3-channel formats (e.g., `VK_FORMAT_R8G8B8_UNORM`) cannot be
      renderred/blitted to natively.  The workaround does not apply to
      modifiers.
    - finally, it treats modifiers specially
      - `isl_drm_modifier_get_score` returns 0 for unsupported modifiers.
        They are rejected.
      - only `ISL_COLORSPACE_LINEAR`/`ISL_COLORSPACE_SRGB` with
        `ISL_UNORM`/`ISL_SFLOAT` are supporeted
        - `ISL_COLORSPACE_YUV` is rejected
        - `ISL_SNORM`/`ISL_UINT`/etc are rejected
      - compressed formats are rejected
      - npot formats (e.g., 3-channel formats) are rejected
      - no disjoint support
      - if planar,
        - only `VK_FORMAT_G8_B8R8_2PLANE_420_UNORM` and
          `VK_FORMAT_G8_B8_R8_3PLANE_420_UNORM` are supported
        - no aux support
      - the only supported aux surface is `CCS_E`
  - `isl_drm_modifier_get_plane_count` is simple
    - each plane has 0 (no compression), 1 (compression), or 2 (compression +
      clear color) aux planes
    - the memory plane count is the format plane count times 1, 2, or 3,
      depending on the modifier
    - a 3-plane formats can in theory has up to 9 memory planes
      - `VK_IMAGE_ASPECT_MEMORY_PLANE_x_BIT_EXT` supports up to 4 memory
        planes
      - what this means is, `vkGetImageSubresourceLayout` or
        `vkGetImageMemoryRequirements2` can query up to 4 memory planes
      - if an image with such a format/modifier is created,
        - it cannot support `VkExternalMemoryImageCreateInfo` in general,
          because we can't query the layouts of all 9 meomry planes which are
          needed by the foreign apis
        - it cannt support `VK_IMAGE_CREATE_DISJOINT_BIT`, which requires
          querying memory requirements for all 9 memory planes
        - but it can still be supported when non-disjoint and non-external
- `anv_GetPhysicalDeviceImageFormatProperties`
  - it calls `anv_get_image_format_properties`
  - only `VK_IMAGE_TILING_OPTIMAL` supports msaa
  - if modifier,
    - only simple 2D image is supported
      - `VK_IMAGE_TYPE_2D`
      - array size 1
      - mip level 1
      - sample count 1
    - the only supported aux suface is `CCS_E`
    - `VK_IMAGE_CREATE_DISJOINT_BIT` plus aux is rejected
    - `VK_IMAGE_CREATE_ALIAS_BIT` is rejected

## `VK_KHR_sampler_ycbcr_conversion`

- spec
  - when a multi-planar format (including single-plane but sub-sampled
    formats) is sampled as `VK_IMAGE_ASPECT_COLOR_BIT`,
    `VkSamplerYcbcrConversion` must be specified
    - when creating the image view
    - when creating the sampler
  - the sampler must be known to pipeline creation through combined image
    sampler and `pImmutableSamplers`
- anv
  - `vk_ycbcr_conversion` is from the runtime
  - `anv_image_view` does not care about `vk_ycbcr_conversion`
  - `anv_sampler` embeds a `vk_sampler` which has `ycbcr_conversion`
  - when a pipeline is created, `nir_vk_lower_ycbcr_tex` lowers the sampling
    if the immutable sampler has a ycbcr conversion
    - this is guaranteed by the spec
    - `lookup_ycbcr_conversion` returns the ycbcr conversion to the lowering
      pass
- `anv_get_format` uses `ycbcr_formats` table for ycbcr formats
  - for `_nPLANE` ones, each plane gets mapped to a regular plane format
    - despite there appear to be some hw support such as
      `ISL_FORMAT_PLANAR_420_8`
    - `nir_vk_lower_ycbcr_tex` lowers the sampling
      - to sample all planes
      - to reconstruct chroma samples 
      - to convert colorspaces
  - for `VK_FORMAT_G8B8G8R8_422_UNORM` and `VK_FORMAT_B8G8R8G8_422_UNORM`,
    they get mapped to hw `ISL_FORMAT_YCRCB_NORMAL` and
    `ISL_FORMAT_YCRCB_SWAPY`
    - `isl_format_is_yuv` returns true in this case
    - `nir_vk_lower_ycbcr_tex` is still used to lower
      - I guess the hw is only responsible for chroma sample reconstruction

## perf

- `intel_perf_new` allocs a `intel_perf_config`
- `intel_perf_init_metrics` initializes the cfg
  - `intel_perf_init_metrics` adds `INTEL_PERF_QUERY_FIELD_TYPE_MI_RPC` reg
  - `oa_metrics_available` checks if oa is available
  - `load_oa_metrics` adds queries
    - `get_register_queries_function` returns the gen-specific function
      - `mtlgt2_register_render_basic_counter_query` adds `RenderBasic`
        queries
      - it is generated from `perf/oa-mtlgt2.xml`
        - `GpuTime` is gpu time elapsed (ns)
        - `GpuCoreClocks` is gpu clock elapsed (cycles)
        - `AvgGpuCoreFrequency` is `GpuCoreClocks * 1000000000 / GpuTime` (hz)
        - `GpuBusy` is `gpu-busy-cycles * 100 / GpuCoreClocks` (%)
- `intel_perf_open`
- `intel_perf_close`

## WAs

- `intel_device_info` is initialized by `intel_get_device_info_from_fd`
  - the pci bus info is from `DRM_DEVICE_GET_PCI_REVISION`
  - the static dev info is from `intel_device_info_init_common` which maps pci
    id to statically-defined tables
    - e.g., `8086:7d45` maps to `intel_device_info_mtl_h`
- `intel_device_info_init_was` initializes WAs
  - `intel_device_info_wa_stepping` returns one of
    - `INTEL_STEPPING_A0`
    - `INTEL_STEPPING_B0`
    - `INTEL_STEPPING_C0`
    - `INTEL_STEPPING_RELEASE`
    - on mtl, it only returns `A0` or `B0`
  - once the stepping is known, `devinfo->workarounds` bitmask is initialized
- `intel_needs_workaround` checks against `devinfo->workarounds`
