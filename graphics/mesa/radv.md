Mesa RADV
=========

## Environment Variables

- `RADV_DEBUG` for radv debug flags
  - `RADV_DEBUG=hang` dumps a report after a GPU hang
    - `radv_check_gpu_hangs` is called after each queue submit.  If the submit
      hangs, a report is written to `$HOME/radv_dumps`
  - `RADV_DEBUG=shaders` dumps shader disassembly
- `RADV_PERFTEST` for radv perf flags
- `ACO_DEBUG` for compiler debug flags
- `RADV_THREAD_TRACE*` generates SQTT trace data for RGP (radeon gpu profiler)
  - `RADV_THREAD_TRACE=100` dumps the SQTT trace for frame #100
    - `radv_handle_sqtt` is called after each successful `sqtt_QueuePresentKHR`
      - `radv_begin_sqtt` is called before frame #100
      - `radv_end_sqtt` is called after frame #100
      - `radv_get_sqtt_trace` is called to retrieve the trace
      - `ac_dump_rgp_capture` saves the trace to `/tmp/*.rgp`
- `RADV_RRA_TRACE*` generates trace data for RRA (radeon raytrace
  analyzer)
- `MESA_VK_MEMORY_TRACE*` generates trace data for RMV (radeon memory
  visualizer)
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

## Tools

- umr
  - <https://gitlab.freedesktop.org/tomstdenis/umr.git>
- RGP

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

## Pipelines

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
- `aco_compile_shader`
  - `aco::select_program` translates nir to aco ir
  - `aco_postprocess_shader` schedules aco ir, performs reg alloc, optimizes,
    eliminates pseudo ops, etc.
  - `aco::emit_program` emits the binary
  - `radv_aco_build_shader_binary` allocs a `radv_shader_binary_legacy` to
    hold the binary
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
