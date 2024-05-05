DRM amdgpu
==========

## Abbreviations

- `cz_ih.c` is Carrizo Interrupt Handler
- `amdgpu_amdkfd.c` is Kernel Fusion Driver for HSA
- `amdgpu_ras.c` is Reliability, Availability, Serviceability
- `si.c` is Southern Islands (GFX6)
  - `AMDGPU_FAMILY_SI`
- `cik.c` is Sea Islands (GFX7)
  - `AMDGPU_FAMILY_CI`
  - `AMDGPU_FAMILY_KV` (apu)
- `vi.c` is Volcanic Islands (GFX8)
- engines
  - DCE is Display and Compositing Engine
  - DCN is Display Core Next
  - UVD is Unified Video Decoder
  - VCE is Video Compression Engine
  - VCN is Video Core Next

## Identifications

- identifications until mid-2021
  - `amdgpu_drv.c` maintains `pciidlist` to map pci ids to `amd_asic_type`
  - `amdgpu_device_ip_early_init` maps `amd_asic_type` to `family`
  - `AMDGPU_FAMILY_SI` is Southern Islands (SI, GFX6)
  - `AMDGPU_FAMILY_CI` and `AMDGPU_FAMILY_KV` (apu) are Sea Islands (CIK,
    GFX7)
  - `AMDGPU_FAMILY_VI` and `AMDGPU_FAMILY_CZ` (apu) are Volcanic Islands (VI,
    GFX8)
  - `AMDGPU_FAMILY_AI` and `AMDGPU_FAMILY_RV` (apu) are Vega (GFX9)
- identifications since mid-2021
  - `amdgpu_drv.c` maintains `pciidlist` to map new amd
    `PCI_CLASS_DISPLAY_VGA` class devices to `CHIP_IP_DISCOVERY` to automatic
    discovery
  - `AMDGPU_FAMILY_NV` and `AMDGPU_FAMILY_VGH`, `AMDGPU_FAMILY_YC`,
    `AMDGPU_FAMILY_GC_10_3_6`, `AMDGPU_FAMILY_GC_10_3_7` (apu) are Navi 1x and
    2x (GFX10)
  - `AMDGPU_FAMILY_GC_11_0_0` and `AMDGPU_FAMILY_GC_11_0_1` are are Navi 3x
    (GFX11)

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

## IP Block Discovery

- `amdgpu_device_ip_early_init` adds the ip blocks
  - `si_set_ip_blocks` is for gfx6 (gcn1, southern islands)
  - `cik_set_ip_blocks` is for gfx7 (gcn2, sea islands)
  - `vi_set_ip_blocks` is for gfx8 (gcn3, volcanic islands, and gcn4, polaris)
  - `amdgpu_discovery_set_ip_blocks` is for gfx9+ (gcn5, vega, all rdnas)
  - `amdgpu_device_ip_block_add` adds an ip block
  - each IP block goes through `early_init`, `sw_init`, `hw_init`, and
    `late_init`
- `enum amd_ip_block_type`
  - `AMD_IP_BLOCK_TYPE_COMMON`: GPU Family
  - `AMD_IP_BLOCK_TYPE_GMC`: Graphics Memory Controller
  - `AMD_IP_BLOCK_TYPE_IH`: Interrupt Handler
  - `AMD_IP_BLOCK_TYPE_SMC`: System Management Controller
  - `AMD_IP_BLOCK_TYPE_PSP`: Platform Security Processor
  - `AMD_IP_BLOCK_TYPE_DCE`: Display and Compositing Engine
  - `AMD_IP_BLOCK_TYPE_GFX`: Graphics and Compute Engine
  - `AMD_IP_BLOCK_TYPE_SDMA`: System DMA Engine
  - `AMD_IP_BLOCK_TYPE_UVD`: Unified Video Decoder
  - `AMD_IP_BLOCK_TYPE_VCE`: Video Compression Engine
  - `AMD_IP_BLOCK_TYPE_ACP`: Audio Co-Processor
  - `AMD_IP_BLOCK_TYPE_VCN`: Video Core/Codec Next
  - `AMD_IP_BLOCK_TYPE_MES`: Micro-Engine Scheduler
  - `AMD_IP_BLOCK_TYPE_JPEG`: JPEG Engine
  - `AMD_IP_BLOCK_TYPE_VPE`: Video Processing Engine
  - `AMD_IP_BLOCK_TYPE_UMSCH_MM`: User Mode Schduler for Multimedia
- gfx6
  - `AMD_IP_BLOCK_TYPE_COMMON`
    - `si_common_ip_block`
  - `AMD_IP_BLOCK_TYPE_GMC`
    - `gmc_v6_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_IH`
    - `si_ih_ip_block`
  - `AMD_IP_BLOCK_TYPE_SMC`
    - `si_smu_ip_block`
  - `AMD_IP_BLOCK_TYPE_DCE`
    - `dce_v6_0_ip_block` (legacy)
    - `dm_ip_block`
  - `AMD_IP_BLOCK_TYPE_GFX`
    - `gfx_v6_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_SDMA`
    - `si_dma_ip_block`
  - `AMD_IP_BLOCK_TYPE_UVD`
    - `uvd_v3_1_ip_block`
- gfx7
  - `AMD_IP_BLOCK_TYPE_COMMON`
    - `cik_common_ip_block`
  - `AMD_IP_BLOCK_TYPE_GMC`
    - `gmc_v7_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_IH`
    - `cik_ih_ip_block`
  - `AMD_IP_BLOCK_TYPE_SMC`
    - `kv_smu_ip_block`
    - `pp_smu_ip_block`
  - `AMD_IP_BLOCK_TYPE_DCE`
    - `dce_v8_1_ip_block` (legacy)
    - `dce_v8_2_ip_block` (legacy)
    - `dce_v8_3_ip_block` (legacy)
    - `dce_v8_5_ip_block` (legacy)
    - `dm_ip_block`
  - `AMD_IP_BLOCK_TYPE_GFX`
    - `gfx_v7_1_ip_block`
    - `gfx_v7_2_ip_block`
    - `gfx_v7_3_ip_block`
  - `AMD_IP_BLOCK_TYPE_SDMA`
    - `cik_sdma_ip_block`
  - `AMD_IP_BLOCK_TYPE_UVD`
    - `uvd_v4_2_ip_block`
  - `AMD_IP_BLOCK_TYPE_VCE`
    - `vce_v2_0_ip_block`
- gfx8
  - `AMD_IP_BLOCK_TYPE_COMMON`
    - `vi_common_ip_block`
  - `AMD_IP_BLOCK_TYPE_GMC`
    - `gmc_v7_4_ip_block`
    - `gmc_v8_0_ip_block`
    - `gmc_v8_1_ip_block`
    - `gmc_v8_5_ip_block`
  - `AMD_IP_BLOCK_TYPE_IH`
    - `cz_ih_ip_block`
    - `iceland_ih_ip_block`
    - `tonga_ih_ip_block`
  - `AMD_IP_BLOCK_TYPE_SMC`
    - `pp_smu_ip_block`
  - `AMD_IP_BLOCK_TYPE_DCE`
    - `dce_v10_0_ip_block` (legacy)
    - `dce_v10_1_ip_block` (legacy)
    - `dce_v11_0_ip_block` (legacy)
    - `dce_v11_2_ip_block` (legacy)
    - `dm_ip_block`
  - `AMD_IP_BLOCK_TYPE_GFX`
    - `gfx_v8_0_ip_block`
    - `gfx_v8_1_ip_block`
  - `AMD_IP_BLOCK_TYPE_SDMA`
    - `sdma_v2_4_ip_block`
    - `sdma_v3_0_ip_block`
    - `sdma_v3_1_ip_block`
  - `AMD_IP_BLOCK_TYPE_UVD`
    - `uvd_v5_0_ip_block`
    - `uvd_v6_0_ip_block`
    - `uvd_v6_2_ip_block`
    - `uvd_v6_3_ip_block`
  - `AMD_IP_BLOCK_TYPE_VCE`
    - `vce_v3_0_ip_block`
    - `vce_v3_1_ip_block`
    - `vce_v3_4_ip_block`
  - `AMD_IP_BLOCK_TYPE_ACP`
    - `acp_ip_block`
- gfx9
  - `AMD_IP_BLOCK_TYPE_COMMON`
    - `vega10_common_ip_block`
  - `AMD_IP_BLOCK_TYPE_GMC`
    - `gmc_v9_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_IH`
    - `vega10_ih_ip_block`
    - `vega20_ih_ip_block`
  - `AMD_IP_BLOCK_TYPE_SMC`
    - `pp_smu_ip_block`
    - `smu_v11_0_ip_block`
    - `smu_v12_0_ip_block`
    - `smu_v13_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_DCE`
    - `dm_ip_block`
  - `AMD_IP_BLOCK_TYPE_GFX`
    - `gfx_v9_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_SDMA`
    - `sdma_v4_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_UVD`
    - `uvd_v7_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_VCE`
    - `vce_v4_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_PSP`
    - `psp_v3_1_ip_block`
    - `psp_v10_0_ip_block`
    - `psp_v11_0_ip_block`
    - `psp_v12_0_ip_block`
    - `psp_v13_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_VCN`
    - `vcn_v1_0_ip_block`
    - `vcn_v2_0_ip_block`
    - `vcn_v2_5_ip_block`
    - `vcn_v2_6_ip_block`
  - `AMD_IP_BLOCK_TYPE_JPEG`
    - `jpeg_v2_0_ip_block`
    - `jpeg_v2_5_ip_block`
    - `jpeg_v2_6_ip_block`
- gfx10 / gfx10.3
  - `AMD_IP_BLOCK_TYPE_COMMON`
    - `nv_common_ip_block`
  - `AMD_IP_BLOCK_TYPE_GMC`
    - `gmc_v10_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_IH`
    - `navi10_ih_ip_block`
  - `AMD_IP_BLOCK_TYPE_SMC`
    - `smu_v11_0_ip_block`
    - `smu_v13_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_DCE`
    - `dm_ip_block`
  - `AMD_IP_BLOCK_TYPE_GFX`
    - `gfx_v10_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_SDMA`
    - `sdma_v5_0_ip_block`
    - `sdma_v5_2_ip_block`
  - `AMD_IP_BLOCK_TYPE_PSP`
    - `psp_v11_0_8_ip_block`
    - `psp_v11_0_ip_block`
    - `psp_v13_0_ip_block`
    - `psp_v13_0_4_ip_block`
  - `AMD_IP_BLOCK_TYPE_VCN`
    - `vcn_v2_0_ip_block`
    - `vcn_v3_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_MES`
    - `mes_v10_1_ip_block`
  - `AMD_IP_BLOCK_TYPE_JPEG`
    - `jpeg_v2_0_ip_block`
    - `jpeg_v3_0_ip_block`
- gfx11 / gfx11.5
  - `AMD_IP_BLOCK_TYPE_COMMON`
    - `soc21_common_ip_block`
  - `AMD_IP_BLOCK_TYPE_GMC`
    - `gmc_v11_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_IH`
    - `ih_v6_0_ip_block`
    - `ih_v6_1_ip_block`
  - `AMD_IP_BLOCK_TYPE_SMC`
    - `smu_v14_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_DCE`
    - `dm_ip_block`
  - `AMD_IP_BLOCK_TYPE_GFX`
    - `gfx_v11_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_SDMA`
    - `sdma_v6_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_PSP`
    - `psp_v14_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_VCN`
    - `vcn_v4_0_ip_block`
    - `vcn_v4_0_3_ip_block`
    - `vcn_v4_0_5_ip_block`
  - `AMD_IP_BLOCK_TYPE_MES`
    - `mes_v11_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_JPEG`
    - `jpeg_v4_0_ip_block`
    - `jpeg_v4_0_3_ip_block`
    - `jpeg_v4_0_5_ip_block`
  - `AMD_IP_BLOCK_TYPE_VPE`
    - `vpe_v6_1_ip_block`
  - `AMD_IP_BLOCK_TYPE_UMSCH_MM`
    - `umsch_mm_v4_0_ip_block`
- gfx12
  - `AMD_IP_BLOCK_TYPE_COMMON`
  - `AMD_IP_BLOCK_TYPE_GMC`
  - `AMD_IP_BLOCK_TYPE_IH`
    - `ih_v7_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_SMC`
  - `AMD_IP_BLOCK_TYPE_DCE`
  - `AMD_IP_BLOCK_TYPE_GFX`
  - `AMD_IP_BLOCK_TYPE_SDMA`
  - `AMD_IP_BLOCK_TYPE_PSP`
  - `AMD_IP_BLOCK_TYPE_VCN`
    - `vcn_v5_0_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_MES`
  - `AMD_IP_BLOCK_TYPE_JPEG`
    - `jpeg_v5_0_0_ip_block`
  - `AMD_IP_BLOCK_TYPE_VPE`
  - `AMD_IP_BLOCK_TYPE_UMSCH_MM`

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
  - it attemps `amdgpu_ring_soft_recovery` first to soft-recover the ring
    - dmesg `ring %s timeout, but soft recovered` if soft-recovered
    - otherwise, `ring %s timeout, signaled seq=%u, emitted seq=%u`
  - it calls `amdgpu_device_should_recover_gpu` to see if it should recover
    the ring
    - dmesg `GPU recovery disabled.` if recovery is disabled
    - it is disabled for `CHIP_STONEY` and some other chips by default
    - set `amdgpu_gpu_recovery` param to override
  - `amdgpu_device_gpu_recover` recovers the GPU
    - if recovery fails, dmesg `GPU Recovery Failed: %d`
- `amdgpu_device_gpu_recover` recovers the gpu after a hang
  - dmesg `amdgpu: GPU reset begin!`
  - `amdgpu_device_pre_asic_reset` calls `amdgpu_device_ip_suspend` to suspend
    all ip blocks
  - `amdgpu_do_asic_reset` resets the asic and resumes
    - `amdgpu_asic_reset` resets the asic
      - dmesg `MODE2 reset`
    - `amdgpu_device_asic_init` posts the asic after reset
      - dmesg `GPU reset succeeded, trying to resume`
    - `amdgpu_device_ip_resume_phase1` resumes some ip blocks
    - `amdgpu_coredump` is added since v6.0
      - it calls `dev_coredumpm` to generate a coredump under
        `/sys/class/devcoredump/devcd*/data`
      - reading the sysfs node calls `amdgpu_devcoredump_read`
      - writing the sysfs node (or after 5 minutes, `DEVCD_TIMEOUT`) deletes
        the device
    - `amdgpu_device_ip_resume_phase2` resumes more ip blocks
    - `amdgpu_ib_ring_tests` tests rings after resume
      - dmesg `IB test failed on %s (%d).` if a ring fails
  - dmesg `GPU reset(%d) succeeded!`
    - or if reset failed, `GPU reset(%d) failed`
  - if reset fails, dmesg `GPU reset end with ret = %d` before the function
    returns
- it looks like amdgpu does not support post-mortem-style hang reports
  - use `RADV_DEBUG=hang` to dumps gpu states to home dir
    - it works better with <https://gitlab.freedesktop.org/tomstdenis/umr>
      installed
- `/sys/kernel/debug/dri/0`
  - `amdgpu_gpu_recover` triggers a gpu reset
    - reading the file schedules `amdgpu_debugfs_reset_work`
  - `amdgpu_reset_dump_register_list` is the list of regs to dump on gpu reset
    - `echo 0x123 0x456 > amdgpu_reset_dump_register_list` to update the list

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
  - it is for userspace export/import
  - `drm_amdgpu_gem_metadata` payload has
    - `__u64 flags` is reserved
    - `__u64 tiling_info` is the tiling flags
      - DCE looks at this, likely because the userspace does not handle it
    - `__u32 data[64]` is completely opaque
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

## `DRM_IOCTL_AMDGPU_CS`

- `drm_amdgpu_cs_in`
  - `ctx_id` is the context
  - `bo_list_handle` and `flags` are no longer used nowadays
  - `chunks` and `num_chunks` point to an array of `drm_amdgpu_cs_chunk`
  - each `drm_amdgpu_cs_chunk` has
    - `chunk_id` is the data type
    - `length_dw` is the data size
    - `chunk_data` points to the data
- radv submits using `amdgpu_cs_submit_raw2` helper
  - there is a `drm_amdgpu_cs_chunk_ib` chunk for each IB
    - `flags` is `AMDGPU_IB_FLAG_x` such as `AMDGPU_IB_FLAG_PREEMPT`
    - `va_start` is the ib addr
    - `ib_bytes` is the ib size
    - `ip_type` is `AMDGPU_HW_IP_x` such as `AMDGPU_HW_IP_GFX`
      - umd uses `AMD_IP_x` such as `AMD_IP_GFX`
    - `ip_instance` is always 0
    - `ring` is the ring idx of the ip type
      - e.g., `AMD_IP_COMPUTE` has 4 or more rings
  - there is a `drm_amdgpu_cs_chunk_fence` chunk for gfx/compute submit
    - `handle` is the bo
    - `offset` is the offset
    - this tells the kernel to write 4 fence seqnos to the offset for
      userspace fencing
  - there is an array of `drm_amdgpu_cs_chunk_syncobj` chunks, one for each
    syncobj wait
    - `handle` is the syncobj
    - `flags` is `DRM_SYNCOBJ_WAIT_FLAGS_x` such as
      `DRM_SYNCOBJ_WAIT_FLAGS_WAIT_ALL`
    - `point` is the timeline value
  - there is an array of `drm_amdgpu_cs_chunk_syncobj` chunks, one for each
    syncobj signal
  - there is a `drm_amdgpu_bo_list_in`
    - `operation` and `list_handle` are ~0
      - it is always assumed to be `AMDGPU_BO_LIST_OP_CREATE`
    - `bo_number`, `bo_info_size`, and `bo_info_ptr` are an array of
      `drm_amdgpu_bo_list_entry` to specify the bos
- radeonsi also submits using `amdgpu_cs_submit_raw2` helper
  - comparing to radv, there is some minor differences
  - `drm_amdgpu_cs_chunk_sem` instead of `drm_amdgpu_cs_chunk_syncobj` because
    the syncobjs are binary
  - `drm_amdgpu_cs_chunk_cp_gfx_shadow` for gfx11+
- `amdgpu_cs_parser_init` initializes the parser
  - it looks up the context id
  - `p->sync` is initialized by `amdgpu_sync_create`
  - `p->exec` is initialized by `drm_exec_init`
- `amdgpu_cs_pass1` copies all chunks and parses some of them
  - `amdgpu_cs_p1_ib` initializes `p->entities`, `p->gang_size`, etc. from
    `drm_amdgpu_cs_chunk_ib`
  - `amdgpu_cs_p1_user_fence` initializes `p->uf_bo` from
    `drm_amdgpu_cs_chunk_fence`
  - `amdgpu_cs_p1_bo_handles` initializes `p->bo_list` from
    `drm_amdgpu_bo_list_in`
  - `p->jobs` is also initialized
- `amdgpu_cs_pass2` parses more chunks
  - `amdgpu_cs_p2_ib` initializes `p->jobs[x]->ibs[y]` from
    `drm_amdgpu_cs_chunk_ib`
  - `amdgpu_cs_p2_syncobj_timeline_wait` adds wait fences from from
    `drm_amdgpu_cs_chunk_syncobj`
    - fences are added to `p->sync`
  - `amdgpu_cs_p2_syncobj_timeline_signal` initializes `p->post_deps` from
    `drm_amdgpu_cs_chunk_syncobj`
- `amdgpu_cs_parser_bos` parses `p->bo_list`
  - `amdgpu_ttm_tt_get_user_pages` gets the user pages for userptr bos
  - `drm_exec_prepare_obj` locks each gem bo for access
  - `amdgpu_ttm_tt_set_user_pages` sets the user pages for userptr bos
  - `amdgpu_cs_bo_validate` validate all bos
- `amdgpu_cs_patch_jobs` patches job for legacy uvd/vce
- `amdgpu_cs_vm_handling` updates vm
  - `amdgpu_vm_clear_freed` updates the page table for freed bos
  - `amdgpu_vm_handle_moved` updates the page table for moved bos
  - `amdgpu_vm_bo_update` updates the page table for bos
  - `amdgpu_vm_update_pdes` updates the higher-level page table
  - more fences are added to `p->sync`
- `amdgpu_cs_sync_rings`
  - `amdgpu_ctx_wait_prev_fence` blocks to wait on prev fence
  - `amdgpu_sync_resv` adds all bo resvs to `p->sync`
  - `amdgpu_sync_push_to_job` adds `p->sync` as the dep of a job
- `amdgpu_cs_submit` submits the jobs
  - `drm_sched_job_arm` arms all jobs
  - `p->fence` is set to the finished fence
  - `ttm_bo_move_to_lru_tail_unlocked` marks the bos used (so that they are
    less likely to be evicted)
  - `amdgpu_ctx_add_fence` adds `p->fence` to `p->entities[x]`
  - `amdgpu_cs_post_dependencies` updates fences for signaled syncobjs
  - `drm_sched_entity_push_job` submits all jobs
- `amdgpu_cs_parser_fini` cleans up

## Display

- display ip blocks
  - gfx6 uses `si_set_ip_blocks` and DCE 6.x
  - gfx7 uses `cik_set_ip_blocks` and DCE 8.x
  - gfx8 uses `vi_set_ip_blocks` and DCE 10.x/11.x
  - gfx9+ uses `amdgpu_discovery_set_display_ip_blocks`
  - newer chips use `dm_ip_block` rather than `dce_v*_ip_block`
- `DRM_IOCTL_MODE_ADDFB2` calls `drm_mode_addfb2_ioctl`
  - it calls `amdgpu_display_user_framebuffer_create` on amdgpu
  - `amdgpu_display_get_fb_info` extracts the tiling from the bo metadata
  - `convert_tiling_flags_to_modifier` converts tiling to modifiers
- `add_gfx9_modifiers`
  - `TILE_VERSION` is `AMD_FMT_MOD_TILE_VER_GFX9`
  - `TILE` is 
    - `AMD_FMT_MOD_TILE_GFX9_64K_D_X`
    - `AMD_FMT_MOD_TILE_GFX9_64K_D`
    - `AMD_FMT_MOD_TILE_GFX9_64K_S_X` (raven only)
    - `AMD_FMT_MOD_TILE_GFX9_64K_S` (raven only)
  - if `TILE` is `*_X`
    - `PIPE_XOR_BITS` depends on `num_pipes` and `num_se`
    - `BANK_XOR_BITS` additionally depends on `num_banks`
  - raven additionally supports `DCC`
    - `TILE` must be `AMD_FMT_MOD_TILE_GFX9_64K_S_X`
    - `DCC_INDEPENDENT_64B` is set
    - `DCC_MAX_COMPRESSED_BLOCK` is `AMD_FMT_MOD_DCC_BLOCK_64B`
    - if `DCC_RETILE`,
      - `RB` depends on `num_se` and `num_rb_per_se`
      - `PIPE` depends on `num_pipes`
  - raven2 additionally supports `DCC_CONSTANT_ENCODE`
- `add_gfx10_1_modifiers`
  - `TILE_VERSION` is `AMD_FMT_MOD_TILE_VER_GFX10`
    - `TILE` is
      - `AMD_FMT_MOD_TILE_GFX9_64K_R_X`
      - `AMD_FMT_MOD_TILE_GFX9_64K_S_X`
    - `PIPE_XOR_BITS` depends on `num_pipes`
    - if `DCC`
      - `TILE` must be `AMD_FMT_MOD_TILE_GFX9_64K_R_X`
      - `DCC_CONSTANT_ENCODE` is set
      - `DCC_INDEPENDENT_64B` is set
      - `DCC_MAX_COMPRESSED_BLOCK` is `AMD_FMT_MOD_DCC_BLOCK_64B`
      - `DCC_RETILE` is optional
  - `TILE_VERSION` is `AMD_FMT_MOD_TILE_VER_GFX9`
    - `TILE` is
      - `AMD_FMT_MOD_TILE_GFX9_64K_D`
      - `AMD_FMT_MOD_TILE_GFX9_64K_S`

## PM

- `amdgpu_pm_ops` is the PM ops
  - `amdgpu_pmops_suspend` sets one of `adev->in_s0ix` or `adev->in_s3` and
    calls `amdgpu_device_suspend` which sets `adev->in_suspend`
  - `amdgpu_pmops_freeze` sets `adev->in_s4 = true` and calls
    `amdgpu_device_suspend` too
- when `echo foo > /sys/power/state`, `state_store` from
  `kernel/power/main.c` is called
  - `disk` is mapped to `PM_SUSPEND_MAX` (S4)
  - `mem` is mapped to `PM_SUSPEND_MEM` first
    - it then gets remapped using `mem_sleep_current`, which usually results
      in `PM_SUSPEND_TO_IDLE` (S0ix) or `PM_SUSPEND_MEM` (S3)
- `amdgpu_acpi_is_s0ix_active` returns true when the followings are all true
  - the gpu is integrated,
  - `pm_suspend_target_state` is S0ix
  - `adev->asic_type` is at least `CHIP_RAVEN`
  - `adev->pm.pp_feature` has `PP_GFXOFF_MASK`
  - acpi `FDAT` has `ACPI_FADT_LOW_POWER_S0`
- `amdgpu_acpi_is_s3_active` returns true when
  - the gpu is discrete, or
  - the gpu is integrated, and `pm_suspend_target_state` is S3
- `amdgpu_acpi_should_gpu_reset` returns true when
  - the gpu is discreted and `pm_suspend_target_state` is S3 or S4, or
  - the gpu is integrated and `pm_suspend_target_state` is S4
