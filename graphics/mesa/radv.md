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
  - bit 22: `RADEON_SURF_DISABLE_DCC` disables compression
  - bit 23: `RADEON_SURF_TC_COMPATIBLE_HTILE` uses TC-compatible HTILE (for
    sampling)
  - bit 24: `RADEON_SURF_IMPORTED` is imported, unused by radv
  - bit 25: `RADEON_SURF_CONTIGUOUS_DCC_LAYERS` is always set
  - bit 26: `RADEON_SURF_SHAREABLE` is shareable, unused by radv
  - bit 27: `RADEON_SURF_NO_RENDER_TARGET` disables render target
  - bit 28: `RADEON_SURF_FORCE_SWIZZLE_MODE` forces swizzle mode, unused by
    radv
  - bit 29: `RADEON_SURF_NO_FMASK` disables FMASK
  - bit 30: `RADEON_SURF_NO_HTILE` disables HTILE (a tiling?)
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
