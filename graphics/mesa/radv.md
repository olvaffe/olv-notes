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
