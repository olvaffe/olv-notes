Mesa RADV
=========

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
- it also sets up 8 memory types
  - 1st is `RADEON_DOMAIN_VRAM` and `RADEON_FLAG_NO_CPU_ACCESS` to
    `RADV_HEAP_VRAM_VIS`
  - 2nd is `RADEON_DOMAIN_GTT`, `RADEON_FLAG_GTT_WC`, and
    `RADEON_FLAG_CPU_ACCESS` to `RADV_HEAP_GTT`
  - 3rd is `RADEON_DOMAIN_VRAM` and `RADEON_FLAG_CPU_ACCESS` to
    `RADV_HEAP_VRAM_VIS`
  - 4th is `RADEON_DOMAIN_GTT` and `RADEON_FLAG_CPU_ACCESS` to
    `RADV_HEAP_GTT`
