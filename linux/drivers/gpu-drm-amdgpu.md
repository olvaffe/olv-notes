DRM amdgpu
==========

## Abbreviations

- `cz_ih.c` is Carrizo Interrupt Handler
- `si.c` is Sea Islands
- `vi.c` is Volcanic Islands

## Initialization

- `amdgpu_init` registers `amdgpu_kms_pci_driver`
  - `amdgpu_pci_probe` calls `amdgpu_driver_load_kms` to inialize the device
- `amdgpu_driver_load_kms` is the main init function
  - `amdgpu_device_init` initializes `struct amdgpu_device`
    - `asic_type` is from `pciidlist` pci table
    - I have `CHIP_STONEY` and `CHIP_RENOIR`
  - `amdgpu_device_ip_early_init` discovers the ip blocks
    - if older like `CHIPSET_RENOIR`
      - `family` is set to `AMDGPU_FAMILY_CZ`
      - `vi_set_ip_blocks` adds some ip blocks 
    - if newer like `CHIP_RENOIR`, `amdgpu_discovery_set_ip_blocks` calls
      `amdgpu_discovery_reg_base_init` to discover the ip blocks
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
