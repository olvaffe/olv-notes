DRM radeon
==========

## Radeon and DMA

- <http://www.botchco.com/agd5f/?p=50>
- A radeon may be on AGP, PCIE, or PCI
  - if the radeon is on AGP and the init failed, it is re-initialized as PCIE or
    PCI
- The MC is programmed to reserve
  - `R_000148_MC_FB_LOCATION` for VRAM
  - `R_00014C_MC_AGP_LOCATION` for AGP when available; the offset into the range
    is added by `R_000170_AGP_BASE` to produce an address in the AGP aperture
- When AGP is not available, the GART is programmed in a way such that an access
  to `RADEON_AIC_LO_ADDR`+`RADEON_AIC_HI_ADDR` is translated using the GART at
  `RADEON_AIC_PT_BASE`.  Similar to AGP.
  - `RADEON_AIC_PT_BASE` is set to the DMA address of the GART table
  - each entry in the gart table is a 32-bit address
  - an entry maps a GPU page number to the DMA address
- In `radeon_check_arguments`, vram is unlimited; gart (`rdev->mc.gtt_size`) is
  limited to 512MB
  - the gart size will be later set to the AGP aperture size if on AGP
- `rdev->mc.mc_vram_size` is set to the size of vram
  - it may be stolen system memory if on IGP
- example
  - VRAM: 1024M 0x0000000000000000 - 0x000000003FFFFFFF (1024M used)
  - GTT: 512M 0x0000000040000000 - 0x000000005FFFFFFF 
  - Detected VRAM RAM=1024M, BAR=256M
- MC
  - VRAM
    - `aper_base` the base address of PCI BAR 0 (mapped VRAM for CPU)
    - `aper_size` the size of PCI BAR 0
    - `visible_vram_size` the size of vram visible to CPU
    - `real_vram_size` the physical size of vram
    - `mc_vram_size` the range to reserve in MC for vram.  Could be larger than
      `real_vram_size`
  - GART
    - `agp_base` the base address of the AGP apertise (for use by GPU)
    - `gtt_size` is set to the AGP apertise size
  - `vram_start` is the start address of VRAM in GPU's view
    - it is 0 for some chipsets; is `aper_base` for others; but it doesn't
      matter
  - `gtt_start` is the start address of GTT in GPU's view
    - it is after `vram_end` for non-AGP; is `agp_base` for AGP
- FB
  - for CPU, vram starts at `mc.aper_base`
  - Say fb bo is at `mc.vram_start + offset`.  It is accesible from CPU at
    `mc.aper_base + offset`, which is fb `smem_start`
- there are `radeon_device` and `drm_radeon_private`
- `resource_copy_region`
- `clear`
- buffer transfer
  - `transfer_inline_write` maps a buffer with DISCARD, memcpy, and unmaps
  - other ops are the trivial
- texture transfer
  - simple except whe the resource is tiled
  - a linear staging texture is created; if for reading, `resource_copy_region`
    is called to copy the tiled texture into the linear texture; if for writing,
    `resource_copy_region` is also called
- `ttm_bo_reserve` is a lockless lock of the bo
