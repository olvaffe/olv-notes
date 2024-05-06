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

## `amdgpu_top`

- build
  - `git clone https://github.com/Umio-Yasuno/amdgpu_top.git`
  - `cargo build`
  - `./target/debug/amdgpu_top`
- <https://github.com/Umio-Yasuno/libdrm-amdgpu-sys-rs.git>
  - it is a wrapper for `libdrm_amdgpu` and amdgpu sysfs and hwmon
  - `DeviceHandle::device_info` queries `AMDGPU_INFO_DEV_INFO`
  - `DeviceHandle::memory_info` queries `AMDGPU_INFO_MEMORY`
  - `DeviceHandle::get_vbios_info` queries `AMDGPU_INFO_VBIOS`
  - `DeviceHandle::sensor_info` calls `amdgpu_query_sensor_info`
  - `DeviceHandle::get_hw_ip_info` calls `amdgpu_query_hw_ip_info` and
    `amdgpu_query_hw_ip_count`
  - `DeviceHandle::get_video_caps_info` calls `amdgpu_query_video_caps_info`
  - `DeviceHandle::query_firmware_version` calls `amdgpu_query_firmware_version`
  - `DeviceHandle::get_min_max_gpu_clock` parses amdgpu `pp_dpm_sclk`
  - `DeviceHandle::get_min_max_memory_clock` parses amdgpu `pp_dpm_mclk`
  - `DeviceHandle::get_gpu_metrics` parses amdgpu `gpu_metrics`
  - `IpDieEntry::get_all_entries_from_sysfs` parses amdgpu `ip_discovery/die/`
  - `PowerProfile::get_all_supported_profiles_from_sysfs` parses amdgpu
    `pp_power_profile_mode`
  - `RasErrorCount::get_from_sysfs_with_ras_block` parses amdgpu
    `ras/*_error_count`
  - `BUS_INFO::get_min_max_link_info_from_dpm` and
    `BUS_INFO::get_current_link_info_from_dpm` parses amdgpu `pp_dpm_pcie`
  - `BUS_INFO::get_max_gpu_link` and `BUS_INFO::get_max_system_link` parses
    pcie `max_link_speed` or `max_link_width`
  - `HwmonTemp` parses hwmon `temp*`
  - `HwmonPower` and `PowerCap` parses hwmon `power*`
- `PpFeatureMask::get_all_enabled_feature` parses
  `/sys/module/amdgpu/parameters/ppfeaturemask`
- `FdInfoStat::get_all_proc_usage` parses `/proc/{pid}/fdinfo/{fd}`
  - <https://docs.kernel.org/gpu/drm-usage-stats.html>
- `AppAmdgpuTop`
  - `new` initializes stats
  - `update` and `update_pc` update stats
