nvtop
=====

## radeontop

- build
  - `git clone https://github.com/clbr/radeontop`
  - `make`
- initialization
  - `init_pci` initializes the hw
    - `find_drm` uses libdrm to find the bus
    - `find_pci` uses libpciaccess to find the pci device
    - `open_drm_bus` opens the primary node
    - `init_amdgpu` sets things up
      - it uses `amdgpu_read_mm_registers` to read regs
        - `getgrbm_amdgpu` reads `GRBM_STATUS`
        - `getsrbm_amdgpu` reads `SRBM_STATUS`
        - `getsrbm2_amdgpu` reads `SRBM_STATUS2`
      - it queries amdgpu info
        - `amdgpu_query_gpu_info` internally queries `AMDGPU_INFO_DEV_INFO`
          - `sclk_max` (shader clock) and `mclk_max` (memory clock) are
            initialized
        - `AMDGPU_INFO_VRAM_GTT` returns vram and gtt sizes
          - `vramsize` and `gttsize` are initialized
      - it uses `amdgpu_query_info` to query more amdgpu info
        - `getvram_amdgpu` queries `AMDGPU_INFO_VRAM_USAGE`
        - `getgtt_amdgpu` queries `AMDGPU_INFO_GTT_USAGE`
        - `getsclk_amdgpu` queries `AMDGPU_INFO_SENSOR_GFX_SCLK`
        - `getmclk_amdgpu` queries `AMDGPU_INFO_SENSOR_GFX_MCLK`
  - `initbits` initializes the bits
- collection
  - `collect` spawns `collector` thread to collect the stats
  - it collects `ticks` samples per second
    - `GRBM_STATUS` indicates which engines are busy
      - when a bit is set, the corresponding engine is busy
    - `SRBM_STATUS` and `SRBM_STATUS2` work the same way
    - the current sclk and mclk in mhz are queried
  - it summarizes every `dumpinterval` seconds
    - the samples are summed (to calculate engine busyness and average clock)
    - the current vram and gtt usage are queried
  - `results` points to the latest summarized data
- ui
  - it uses `halfdelay(10)` and `getch` to update the screen every 1 second
  - it visualizes `results`
