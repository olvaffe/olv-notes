DRM amdgpu
========

## Ryzen 5 4500U

- system info
  - APU
  - with 8GB memory
  - recognized as RENOIR
- in `gmc_v9_0_sw_init`
  - `amdgpu_vm_adjust_size` is called with a VM size of 256TB
  - `gmc_v9_0_mc_init`
    - `mc_vram_size` is 512M, likely carved out from the physical memory
    - `real_vram_size`, `aper_size`, `visible_vram_size` are all set to `mc_vram_size`
    - `gart_size` is hard coded to 1G
    - `amdgpu_gmc_gart_location` picks the gart location to be before or after vram
      - in this case, before
    - `amdgpu_gmc_agp_location` picks the agp location to be before or after vram
      - in this case, after and can use up to 48-bit address space
  - `amdgpu_bo_init`
    - in `amdgpu_ttm_init`, GTT size is set to `AMDGPU_DEFAULT_GTT_SIZE_MB` (3GB)
  - relevant log
    - `[drm] vm size is 262144 GB, 4 levels, block size is 9-bit, fragment size is 9-bit`
    - `amdgpu 0000:04:00.0: amdgpu: VRAM: 512M 0x000000F400000000 - 0x000000F41FFFFFFF (512M used)`
    - `amdgpu 0000:04:00.0: amdgpu: GART: 1024M 0x0000000000000000 - 0x000000003FFFFFFF`
    - `amdgpu 0000:04:00.0: amdgpu: AGP: 267419648M 0x000000F800000000 - 0x0000FFFFFFFFFFFF`
    - `[drm] Detected VRAM RAM=512M, BAR=512M`
    - `[drm] RAM width 128bits DDR4`
    - `[drm] amdgpu: 512M of VRAM memory ready`
    - `[drm] amdgpu: 3072M of GTT memory ready.`
- in `gmc_v9_0_gart_init`
  - GART is enabled
    - it is a 1GB aperture into the 3GB GTT
    - GPU uses a 256TB VM so softpin works
    - address translations: GPU LA -> GPU PA -> MEM PA
  - relevant log
    - `[drm] GART: num cpu pages 262144, num gpu pages 262144`
    - `[drm] PCIE GART of 1024M enabled.`
- radv
  - radv sums visible VRAM (0.5GB) and GTT (3GB) and sets up two fake heaps
    - first is non-local `RADV_HEAP_GTT` heap that gets 1.166GB
    - second is local `RADV_HEAP_VRAM_VIS` heap that gets 2.333GB
  - it also sets up 8 memory types
    - 1st is `RADEON_DOMAIN_VRAM` and `RADEON_FLAG_NO_CPU_ACCESS` to
      `RADV_HEAP_VRAM_VIS`
    - 2nd is `RADEON_DOMAIN_GTT`, `RADEON_FLAG_GTT_WC`, and
      `RADEON_FLAG_CPU_ACCESS` to `RADV_HEAP_GTT`
    - 3rd is `RADEON_DOMAIN_VRAM` and `RADEON_FLAG_CPU_ACCESS` to
      `RADV_HEAP_VRAM_VIS`
    - 4th is `RADEON_DOMAIN_GTT` and `RADEON_FLAG_CPU_ACCESS` to
      `RADV_HEAP_GTT`
