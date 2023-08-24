DRM amdgpu
==========

## Abbreviations

- `cz_ih.c` is Carrizo Interrupt Handler
- `amdgpu_amdkfd.c` is Kernel Fusion Driver for HSA
- `amdgpu_ras.c` is Reliability, Availability, Serviceability
- `si.c` is Sea Islands
- `vi.c` is Volcanic Islands

## Initialization

- `amdgpu_init` registers `amdgpu_kms_pci_driver`
  - `amdgpu_pci_probe` calls `amdgpu_driver_load_kms` to inialize the device
  - `driver_data` is from `pciidlist`, and on latest gens, it is `CHIP_IP_DISCOVERY`
- `amdgpu_driver_load_kms` is the main init function
  - `amdgpu_device_init` initializes `struct amdgpu_device`
    - `asic_type` is from `pciidlist` pci table
    - I have `CHIP_STONEY` and `CHIP_RENOIR`
  - `amdgpu_device_ip_early_init` discovers the ip blocks
    - if older like `CHIPSET_STONEY`
      - `family` is set to `AMDGPU_FAMILY_CZ`
      - `vi_set_ip_blocks` adds some ip blocks 
    - if newer like `CHIP_RENOIR`, `amdgpu_discovery_set_ip_blocks` calls
      `amdgpu_discovery_reg_base_init` to discover the ip blocks
    - `amdgpu_get_bios` reads bios to `adev->bios` from various possible
      locations
      - `adev->is_atom_fw` is set since `CHIP_VEGA10`
  - `amdgpu_device_ip_init` initializes the ip blocks
- IP blocks
  - ip blocks are discovered and added with `amdgpu_device_ip_block_add`
  - each IP block goes through `early_init`, `sw_init`, `hw_init`, and
    `late_init`
  - `AMD_IP_BLOCK_TYPE_COMMON`: GPU Family
    - `CHIPSET_STONEY` has `vi_common_ip_block`
    - `CHIPSET_RENOIR` has `vega10_common_ip_block`
  - `AMD_IP_BLOCK_TYPE_GMC`: Graphics Memory Controller
    - `CHIPSET_STONEY` has `gmc_v8_0_ip_block`
    - `CHIPSET_RENOIR` has `gmc_v9_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_IH`: Interrupt Handler
    - `CHIPSET_STONEY` has `cz_ih_ip_block`
    - `CHIPSET_RENOIR` has `vega10_ih_ip_block`
  - `AMD_IP_BLOCK_TYPE_SMC`: System Management Controller
    - `CHIPSET_STONEY` has `pp_smu_ip_block`
    - `CHIPSET_RENOIR` has `smu_v11_0_ip_block` (or later?)
  - `AMD_IP_BLOCK_TYPE_PSP`: Platform Security Processor
    - `CHIPSET_RENOIR` has `psp_v3_1_ip_block` (or later?)
  - `AMD_IP_BLOCK_TYPE_DCE`: Display and Compositing Engine
    - `CHIPSET_STONEY` has `dm_ip_block`
    - `CHIPSET_RENOIR` has `dm_ip_block`
  - `AMD_IP_BLOCK_TYPE_GFX`: Graphics and Compute Engine
    - `CHIPSET_STONEY` has `gfx_v8_1_ip_block`
    - `CHIPSET_RENOIR` has `gfx_v9_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_SDMA`: System DMA Engine
    - `CHIPSET_STONEY` has `sdma_v3_0_ip_block`
    - `CHIPSET_RENOIR` has `sdma_v4_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_UVD`: Unified Video Decoder
    - `CHIPSET_STONEY` has `uvd_v6_2_ip_block`
  - `AMD_IP_BLOCK_TYPE_VCE`: Video Compression Engine
    - `CHIPSET_STONEY` has `vce_v3_4_ip_block`
  - `AMD_IP_BLOCK_TYPE_ACP`: Audio Co-Processor
    - `CHIPSET_STONEY` has `acp_ip_block`
  - `AMD_IP_BLOCK_TYPE_VCN`: Video Core/Codec Next
    - `CHIPSET_RENOIR` has `vcn_v2_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_MES`: Micro-Engine Scheduler
  - `AMD_IP_BLOCK_TYPE_JPEG`: JPEG Engine
    - `CHIPSET_RENOIR` has `jpeg_v2_0_ip_block`

## GMC

- `AMD_IP_BLOCK_TYPE_GMC` is the graphics memory controller
  - use `gmc_v9_0_sw_init` and `gmc_v9_0_hw_init` with `CHIP_VEGA20` for
    example
  - there are two hubs: GFX (graphics and compute) and MM (sdma, uvd, vce)
  - xGMI is inter-chip global memory interconnect, for mulitple cards to share
    their resources, I guess
  - colloect information
    - `mc_mask` is 48-bit
    - `mc_vram_size` is the VRAM size and is read from `mmRCC_CONFIG_MEMSIZE`
      reg.  `real_vram_size` is set to `mc_vram_size`
    - `aper_base` and `aper_size` is PCI BAR 0.  It provides MMIO access to the
      VRAM.  It is usually 256MiB, and the driver makes an (usually failed)
      attempt to resize it to `mc_vram_size`
    - `visible_vram_size` is set to `aper_size`
    - `gart_size` is fixed at 512M (requires swap in/out when accessing system
      memory)
  - with all these information, we can assign the GPU physical address space
    - `fb_start` and `fb_end` are read from `mmMC_VM_FB_LOCATION_BASE/TOP`,
      and they specify the address range of the VRAM.  In xGMI setup, a card
      can see the VRAMs of any card in the hive
    - `gart_start` and `gart_end` are assigned to either the begin or the end
      of the entire address space.  It is merely 512M.
    - `agp_start` and `agp_size` use the bigger one of the two holes outside
      of fb and gart
  - initialize BO manager
    - mark `aper_base`/`aper_size` WC with MTRR
    - init `TTM_PL_VRAM` of size `real_vram_size`
    - init `TTM_PL_TT` of size the smaller of `mc_vram_size` and 75% of system
      ram 
      - because GART is 512MB, GPU can only see 512MB of it at any point
  - enable GART and others
    - allocate from VRAM a BO to hold the GART table
    - set `mmVM_CONTEXT0_PAGE_TABLE_BASE_ADDR_LO32/HI32` to the BO physical
      address to specify the location of the GART table
    - set `mmVM_CONTEXT0_PAGE_TABLE_START_ADDR_LO32/HI32` to `gart_start`
    - set `mmVM_CONTEXT0_PAGE_TABLE_END_ADDR_LO32/HI32` to `gart_end`
    - set `mmMC_VM_AGP_BOT/TOP` to `agp_start` and `agp_end`
    - set `mmMC_VM_SYSTEM_APERTURE_LOW/HIGH_ADDR` to the used range of the
      physical address space
- each process has its own virtual address space for GPU
  - VM is initialized with `amdgpu_vm_init`
  - `job->vm_pd_addr = amdgpu_gmc_pd_addr(vm->root.base.bo)` and send the
    physical address of the page directly to the engine
- `gmc_v9_0_sw_init` again with `CHIP_RENOIR`
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
    - in `amdgpu_ttm_init`, GTT size is set to `ttm_tt_pages_limit`
      - `ttm_global_init` calls `ttm_tt_mgr_init` to set the limit to half of
        sysram
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
- <https://lore.kernel.org/all/CADnq5_MfSNHP--KyQe2GEWCvg4XAwLV5FAw+0B-0DwvXBcACtw@mail.gmail.com/T/#mb034b7f92c4b3351038c71611d7a6d574fc9bee9>

## PM sysfs

- `amdgpu_pm_sysfs_init` and `amdgpu_debugfs_pm_init`
- `hwmon_device_register_with_groups` with `hwmon_attributes`
  - temperature
    - `temp*_input`: on-die gpu temperature in millidegrees Celcius
    - `temp*_label`: channel label (edge, junction, and mem)
    - `temp*_crit`
    - `temp*_crit_hyst`
    - `temp*_emergency`
  - voltage
    - `in0_input` and `in0_label`: millivolts on gpu
    - `in1_input` and `in1_label`: millivolts on northbridge
  - power
    - `power*_average`: average power used by gpu in micro-watts
    - `power*_cap_max`: max cap
    - `power*_cap_min`: min cap
    - `power*_cap`: selected cap
    - `power*_cap_default`: default cap
    - `power*_label`: requested power (slowPPT or fastPPT)
  - fan
    - `pwm1`: fan level (0-255)
    - `pwm1_enable`: fan control (0 is no control; 1 manual; 2 auto)
    - `pwm1_min`: min level (0)
    - `pwm1_max`: max level (0)
    - `fan1_input`: cur rpm
    - `fan1_min`: min rpm
    - `fan1_max`: max rpm
    - `fan1_target`: target rpm
    - `fan1_enable`: enable or disable sensor
  - frequency
    - `freq*_input`: gpu (sclk) or mem (mclk) freq in hz
    - `freq*_label`: sclk or mclk
- `amdgpu_device_attr_create_groups` with `amdgpu_device_attrs`
  - `power_dpm_state`: legacy interface
  - `power_dpm_force_performance_level`
    - `auto`
    - `low` forces clocks to lowest freqs
    - `high` forces clocks to highest freqs
    - `manual` allows manual adjs through `pp_dpm_{mclk,sclk,pcie}` and
      `pp_power_profile_mode`
    - `profile_*` disables clock gating and is for profiling
  - `pp_num_states`, `pp_cur_state`, and `pp_force_state`
    - default, battery, balanced, and performance
    - pp stands for PowerPlay
  - `pp_table` sets or shows the current powerplay table
  - `pp_dpm_sclk`, `pp_dpm_mclk`, `pp_dpm_socclk`, `pp_dpm_fclk`,
    `pp_dpm_vclk`, `pp_dpm_dclk`, `pp_dpm_dcefclk`, `pp_dpm_pcie`
    - shows available power levels in the current power state and the freq
    - set `power_dpm_force_performance_level` to `manual` to set
  - `pp_sclk_od`
  - `pp_mclk_od`
  - `pp_power_profile_mode` shows/sets power profiles
    - `3D_FULL_SCREEN`, `VIDEO`, `VR`, `COMPUTE`, `CUSTOM`
  - `pp_od_clk_voltage`
  - `gpu_busy_percent` shows how busy the gpu is
  - `mem_busy_percent` shows how busy VRAM is
  - `pcie_bw` shows pcie bandwidth used
  - `pp_features` shows/sets powerplay features
  - `unique_id` shows the unique id for the gpu
  - `thermal_throttling_logging` logs thermal throttle
  - `gpu_metrics` gives a snapshot of all gpu metrics (temp, freq,
    utilization, etc)
  - `smartshift_apu_power` shows apu power shift in percent
  - `smartshift_dgpu_power` shows dgpu power shift in percent
  - `smartshift_bias` is between -100 (max for apu) to 100 (max for dgpu)
- `amdgpu_debugfs_pm_init` registers `amdgpu_pm_info`
  - it shows all interesting metrics (freqs, voltage, watts, temp, load,
    gating) in human-readable form
- force a frequency for SCLK,
  - echo `manual` to `power_dpm_force_performance_level`
  - cat `pp_dpm_sclk` to see valid power levels and echo the level to select
    - on renoir, `force_clock_level` points to `smu_force_ppclk_levels` which
      calls `renoir_force_clk_levels` to set `SMU_MSG_SetSoftMaxGfxClk`
      `SMU_MSG_SetHardMinGfxClk`
    - there are only 3 levels: hw max, hw min, and 700
      (`RENOIR_UMD_PSTATE_GFXCLK`)
  - or, cat `pp_od_clk_voltage` and echo to reprogram the table
    - for example,
      - `echo 's 0 1500' > pp_od_clk_voltage`
      - `echo 's 1 1500' > pp_od_clk_voltage`
      - `echo 'c' > pp_od_clk_voltage`
    - on renoir, `set_fine_grain_clk_vol` is NULL, `odn_edit_dpm_table` points
      to `smu_od_edit_dpm_table` which calls `renoir_od_edit_dpm_table` to set
      `SMU_MSG_SetSoftMaxGfxClk` and `SMU_MSG_SetHardMinGfxClk`
    - the values are arbitrary, as long as they are between hw min/max

## GPU Hang

- `amdgpu_job_timedout` is called when a job takes too long and times out
  - if `amdgpu_device_should_recover_gpu` returns true, `amdgpu_gpu_recovery`
    is called to recover the GPU
    - `CHIP_STONEY` does not support recovery?
- `amdgpu_gpu_recovery` recovers the gpu after a hang
  - `amdgpu_do_asic_reset` resets the gpu
    - `amdgpu_reset_capture_coredumpm` is added since v6.0
      - it calls `dev_coredumpm` to generate a coredump
- it looks like amdgpu does not support post-mortem-style hang reports
  - use `RADV_DEBUG=hang` to dumps gpu states to home dir
    - it works better with <https://gitlab.freedesktop.org/tomstdenis/umr>
      installed

## DRM scheduler

- `amdgpu_device_init_schedulers` is called from `amdgpu_device_ip_init` to
  init schedulers
  - this initializes per-ring `drm_gpu_scheduler`
  - on a GFX9 APU, there are these rings
    - 1x `gfx`, initialized by `gfx_v9_0_sw_init`
    - 8x `comp_x.y.z`, initialized by `gfx_v9_0_compute_ring_init`
    - 1x `sdma0`, initialized by `sdma_v4_0_sw_init` I guess
    - 1x `vcn_dec` and 2x `vcn_encX`, initialized by `vcn_v2_0_sw_init` I guess
    - 1x `jpeg_dec`, initialized by `jpeg_v2_0_sw_init`
      - on older kernel this is `vcn_jpeg` and is a part of `vcn_v2_0_sw_init`
- `drm_sched_init` initializes `drm_gpu_scheduler`
  - it starts a kthread running `drm_sched_main`
  - the kthread calls `sched_set_fifo_low` to become `SCHED_FIFO` with low
    static rt priority
  - `sched` is per-ring `ring->sched`
  - `ops` is shared `amdgpu_sched_ops`
  - `hw_submission` is per-ring `ring->num_hw_submission`
    - it seems to default to `amdgpu_sched_hw_submission` (2) for most rings
      and at least 256 for SDMA
  - `hang_limit` is `amdgpu_job_hang_limit`
    - default is 0
  - `timeout` is shared `adev->xxx_timeout`
     - default timeout for compute is 60s (or unlimited on older kernels) and
       10s for others
     - can be changed via `amdgpu.lockup_timeout=...`
  - `timeout_wq` is shared `adev->reset_domain->wq`
  - `score` is per-ring `ring->sched_score`
    - usually NULL unless newer VCN rings
- timeout
  - when `drm_sched_job_begin` begins a job, it also starts a delayed work
    with `drm_sched_start_timeout`
  - if the job takes too long, `drm_sched_job_timedout` removes the job and
    calls back into amdgpu
  - `amdgpu_job_timedout` attemps to do a soft recovery before a hw reset

## ioctls

- `DRM_IOCTL_AMDGPU_INFO` queries various info about a device
  - `AMDGPU_INFO_FW_VERSION` queries firmware versions
    - mesa is interested in `AMDGPU_INFO_FW_GFX_{ME,MEC,PFP}` and
      `AMDGPU_INFO_FW_{UVD,VCE}`
  - `AMDGPU_INFO_HW_IP_INFO` queries ip info
    - `hw_ip_version_*` is from `adev->ip_blocks[x].version`
    - `ip_discovery_version` is from `adev->ip_versions[x][0]` and is used for
      newer gens
    - mesa uses
      `AMD_IP_GFX`/`AMDGPU_HW_IP_GFX`/`AMD_IP_BLOCK_TYPE_GFX`/`GC_HWIP` to
      determine the gen
- `DRM_IOCTL_AMDGPU_CTX` allocs/frees/queries a context
  - radv creates a ctx for each `VkQueue`
  - radv uses `AMDGPU_CTX_OP_QUERY_STATE2` for gpu hang check
    - `AMDGPU_CTX_QUERY2_FLAGS_RESET` and `AMDGPU_CTX_QUERY2_FLAGS_GUILTY` are
      sticky and the context should be re-created
  - radv uses `AMDGPU_CTX_OP_SET_STABLE_PSTATE` to force peak performance in
    `vkAcquireProfilingLockKHR`
- `DRM_IOCTL_AMDGPU_VM` reserves the vmid
  - it is used for shader debugging
- `DRM_IOCTL_AMDGPU_GEM_CREATE` creates a gem bo
- `DRM_IOCTL_AMDGPU_GEM_USERPTR` creates a gem bo from userptr
- `DRM_IOCTL_AMDGPU_GEM_MMAP` returns the magic mmap offset for cpu mapping
- `DRM_IOCTL_AMDGPU_GEM_METADATA` sets/gets the bo metadata
- `DRM_IOCTL_AMDGPU_GEM_OP` queries/updates gem bo info
- `DRM_IOCTL_AMDGPU_GEM_VA` maps/unmaps/replaces gem bo va
- `DRM_IOCTL_AMDGPU_CS` submits a job
  - `drm_amdgpu_cs_out::handle` is the seqno of the job
- `DRM_IOCTL_AMDGPU_WAIT_CS` waits for a job
- somewhat legacy
  - `DRM_IOCTL_AMDGPU_SCHED` changes the priority of any context
    - master only
    - no user?
  - `DRM_IOCTL_AMDGPU_BO_LIST` creates/destroys/updates a bo list
    - deprecated by `DRM_IOCTL_AMDGPU_CS` in favor of
      `AMDGPU_CHUNK_ID_BO_HANDLES`
  - `DRM_IOCTL_AMDGPU_GEM_WAIT_IDLE` waits on the implicit fence of a bo
    - only used by radeonsi
  - `DRM_IOCTL_AMDGPU_FENCE_TO_HANDLE` converts a `drm_amdgpu_fence` to a
    syncobj, syncobj fd, or sync file fd
    - only used by radeonsi
  - `DRM_IOCTL_AMDGPU_WAIT_FENCES` waits an array of `drm_amdgpu_fence`s
    - no user
