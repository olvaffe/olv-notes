Mesa RADV
=========

## Winsys

- `radv_amdgpu_winsys_create`
  - libdrm's `amdgpu_device_initialize` is called to initialize
    `amdgpu_device_handle`
    - `dev->dev_info` is from kernel's `AMDGPU_INFO_DEV_INFO`
    - `dev->vamgr{,_32,_high,_high_32}` are initialized
      - on a renoir apu,
        - `vamgr_32` is `[2MB, 4GB)`
        - `vamgr` is `[4GB, 128TB)` (48-bit address space)
        - `vamgr_high_32` and `vamgr_high` are similarly sized but the higher
          16 bits are all 1's (that is, about 128TB near the end of 64-bit
          address space)
  - `ac_query_gpu_info` is called to initialize `radeon_info`
    - `meminfo` is from kernel's `AMDGPU_INFO_MEMORY`
      - on a renoir apu,
        - `vram.total_heap_size` is 64M (carve-out?)
        - `cpu_accessible_vram.total_heap_size` is also 64M
        - `gtt.total_heap_size` is near 4GB (kernel uses half of usable system
          memory in `amdgpu_ttm_init`)
    - `fix_vram_size` aligns vram size to 256MB

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
  - otherwise, `radv_expand_depth_stencil_compute` resolves using the compute
    pipeline
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
