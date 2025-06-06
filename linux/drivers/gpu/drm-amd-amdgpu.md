DRM amdgpu
==========

## Config

- `CONFIG_DRM_AMDGPU` enables amdgpu
- `CONFIG_DRM_AMDGPU_SI` enables experimental SI (gfx6) support
- `CONFIG_DRM_AMDGPU_CIK` enables experimental CIK (gfx7) support
- `CONFIG_DRM_AMD_ISP` enables ISP support for mipi cameras
  - new ip block on strix halo apu
- `CONFIG_DRM_AMD_DC` enables DC/DM support
  - SI/CIK/VI (gfx6-8) can use DC/DM or legacy path
  - Vega and later (gfx9+) requires DC/DM
- `CONFIG_DRM_AMD_ACP` enables ACP support for i2s audio codecs
  - ip block on VI (gfx8)
- `CONFIG_HSA_AMD` enables HSA driver

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
    - `amdgpu_atombios_init` parses the bios
  - `amdgpu_device_ip_init` initializes the ip blocks
- `amdgpu_get_bios`
  - the legacy `radeon_get_bios` supports
    - `radeon_atrm_get_bios` uses ACPI ATRM
      - this was added with switcheroo, for dgpu on laptops
    - `radeon_acpi_vfct_bios` uses ACPI VFCT
      - this was added for uefi systems
    - `radeon_read_bios` uses `pci_map_rom`
      - this was the orignal method where the vga bios was stored in pci rom
    - `radeon_read_platform_bios` uses `pci_dev::rom`
      - some platforms used this to access pci rom
  - `amdgpu_atrm_get_bios` uses ACPI ATRM
  - `amdgpu_acpi_vfct_bios` uses ACPI VFCT
    - this seems to be the modern way for igpu
  - `igp_read_bios_from_vram` checks the start of vram
  - `amdgpu_read_bios` uses `pci_map_rom`, the pci rom bar
  - `amdgpu_read_bios_from_rom` uses `amdgpu_asic_read_bios_from_rom`
    - this seems to be the modern way for dgpu since gcn5
  - `amdgpu_read_platform_bios` uses `pci_dev::rom`
  - `adev->is_atom_fw` is set since `CHIP_VEGA10`

## IP Block Discovery

- `amdgpu_device_ip_early_init` adds the ip blocks
  - `si_set_ip_blocks` is for gfx6 (gcn1, southern islands)
  - `cik_set_ip_blocks` is for gfx7 (gcn2, sea islands)
  - `vi_set_ip_blocks` is for gfx8 (gcn3, volcanic islands, and gcn4, polaris)
  - `amdgpu_discovery_set_ip_blocks` is for gfx9+ (gcn5, vega, all rdnas)
- `amdgpu_discovery_set_ip_blocks`
  - `amdgpu_discovery_reg_base_init` reads and parses
    `amdgpu/ip_discovery.bin` from gpu memory on rdna+ (and renoir)
    - `hw_id_map` maps `*_HWIP` to `*_HWID`, the real hwid
    - `ip_versions` are initialized
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
- `enum amd_hw_ip_block_type`
  - `GC_HWIP` determines
    - `AMD_IP_BLOCK_TYPE_COMMON`
    - `AMD_IP_BLOCK_TYPE_GMC`
    - `AMD_IP_BLOCK_TYPE_GFX`
    - `AMD_IP_BLOCK_TYPE_MES`
    - `adev->family`
    - `adev->flags |= AMD_IS_APU`
    - `adev->gfxhub.funcs`
    - `adev->gfx.funcs`
  - `HDP_HWIP` determines `adev->hdp.funcs`
  - `SDMA0_HWIP` determines `AMD_IP_BLOCK_TYPE_SDMA`
  - `SDMA[1-7]_HWIP`
  - `LSDMA_HWIP` determines `adev->lsdma.funcs`
  - `MMHUB_HWIP` determines `adev->mmhub.funcs`
  - `ATHUB_HWIP`
  - `NBIO_HWIP` determines `adev->nbio.funcs`
  - `MP0_HWIP` determines `AMD_IP_BLOCK_TYPE_PSP`
  - `MP1_HWIP` determines `AMD_IP_BLOCK_TYPE_SMC`
  - `UVD_HWIP` determines
    - `AMD_IP_BLOCK_TYPE_UVD`
    - `AMD_IP_BLOCK_TYPE_VCN`
    - `AMD_IP_BLOCK_TYPE_JPEG`
    - `AMD_IP_BLOCK_TYPE_UMSCH_MM`
  - `VCN_HWIP` and `JPEG_HWIP` are alternative names for `UVD_HWIP`
  - `VCN1_HWIP`
  - `VCE_HWIP` determines `AMD_IP_BLOCK_TYPE_VCE`
  - `VPE_HWIP` determines `AMD_IP_BLOCK_TYPE_VPE`
  - `DF_HWIP` determines `adev->df.funcs`
  - `DCE_HWIP` determines `AMD_IP_BLOCK_TYPE_DCE`
  - `OSSSYS_HWIP` determines `AMD_IP_BLOCK_TYPE_IH`
  - `SMUIO_HWIP` determines `adev->smuio.funcs`
  - `PWR_HWIP`
  - `NBIF_HWIP`
  - `THM_HWIP`
  - `CLK_HWIP`
  - `UMC_HWIP` determines `adev->umc`
  - `RSMU_HWIP`
  - `XGMI_HWIP` determines `adev->gmc.xgmi`
  - `DCI_HWIP` determines `AMD_IP_BLOCK_TYPE_DCE`
  - `PCIE_HWIP`
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

## Firmwares

- `amdgpu_device_init`
  - `amdgpu_device_check_arguments` `amdgpu_ucode_get_load_type` to initialize
    `adev->firmware.load_type`
    - `AMDGPU_FW_LOAD_SMU` on gfx8
    - `AMDGPU_FW_LOAD_PSP` on gfx9+
  - `amdgpu_device_ip_early_init` adds and early inits the ip blocks
    - `amdgpu_discovery_set_ip_blocks` adds the ip blocks on gfx9+
    - `amd_ip_funcs::early_init` early inits
      - firmwares are usually loaded from disk in this phase
  - `amdgpu_device_ip_init` inits ip blocks
    - `amd_ip_funcs::sw_init` for all ip blocks
    - `amd_ip_funcs::hw_init` for common and gmc ip blocks
    - `amdgpu_ucode_create_bo`
    - `amdgpu_device_ip_hw_init_phase1` for ih ip block
    - `amdgpu_device_fw_loading`
      - `amd_ip_funcs::hw_init` for psp ip block
    - `amdgpu_device_ip_hw_init_phase2`
      - `amd_ip_funcs::hw_init` for the rest
- when `AMDGPU_FW_LOAD_PSP`,
  - `amdgpu_ucode_create_bo` creates the gpu memory for most firmwares
    - `amd_ip_funcs::early_init` has loaded the firmwares from disk and
      incremented `adev->firmware.fw_size`
    - this function allocates the bo to hold the firmwares
  - `psp_hw_init` calls `amdgpu_ucode_init_bo` to copy firmwares to the bo
  - `psp_load_fw` loads the firmwares
    - `psp_hw_start` starts psp ip block
    - `psp_load_non_psp_fw` calls `psp_execute_ip_fw_load` to load the
      firmwares from gpu memory to ip blocks
- loading firmware from disk
  - `amdgpu_ucode_ip_version_decode` returns the fw name prefix
    - `ip_maj_min_rev`, such as `gc_10_3_7`
      - `GC_HWIP` is `gc`
      - `SDMA0_HWIP` is `sdma`
      - `MP0_HWIP` is `psp`
      - `MP1_HWIP` is `smu`
      - `UVD_HWIP` is `vcn`
      - `VPE_HWIP` is `vpe`
    - `amdgpu_ucode_legacy_naming` might return legacy name prefix; e.g.,
      - `SDMA0_HWIP` is `yellow_carp_sdma`
      - `MP0_HWIP` is `yellow_carp`
      - `UVD_HWIP` is `yellow_carp_vcn`
  - `amdgpu_ucode_request` calls `request_firmware` to load the fw from
    filesystem
- gfx10 (non-exhaustive)
  - `AMD_IP_BLOCK_TYPE_DCE`
    - `dm_init_microcode` loads
      - `amdgpu/dcn_3_1_6_dmcub.bin`
    - `dm_dmub_hw_init` loads the fw to the ip block and starts the ip block
  - `AMD_IP_BLOCK_TYPE_GFX`
    - `gfx_v10_0_init_microcode` loads
      - `amdgpu/gc_10_3_7_rlc.bin`
      - `amdgpu/gc_10_3_7_me.bin`
      - `amdgpu/gc_10_3_7_pfp.bin`
      - `amdgpu/gc_10_3_7_ce.bin`
      - `amdgpu/gc_10_3_7_mec.bin`
      - `amdgpu/gc_10_3_7_mec2.bin`
    - `gfx_v10_0_hw_init`
      - `gfx_v10_0_rlc_resume` waits for rlc to start
      - `gfx_v10_0_cp_resume` calls `gfx_v10_0_cp_async_gfx_ring_resume`
  - `AMD_IP_BLOCK_TYPE_SDMA`
    - `amdgpu_sdma_init_microcode` loads
      - `amdgpu/sdma_5_2_7.bin`
    - `sdma_v5_2_start` inits and starts SDMA
  - `AMD_IP_BLOCK_TYPE_PSP`
    - `psp_v13_0_init_microcode` loads
      - `amdgpu/psp_13_0_8_ta.bin`
      - `amdgpu/psp_13_0_8_toc.bin`
    - `psp_hw_start` starts the hw
        - `psp_load_toc` loads toc
    - ta fw seems to consists of multiple binaries
      - `psp_ras_initialize`
      - `psp_hdcp_initialize`
      - `psp_dtm_initialize`
      - `psp_rap_initialize`
      - `psp_securedisplay_initialize`
  - `AMD_IP_BLOCK_TYPE_VCN`
    - `amdgpu_vcn_early_init` loads
      - `amdgpu/yellow_carp_vcn.bin`
    - `nbio_v7_2_vcn_doorbell_range` starts the ip block?

## SOC

- `AMD_IP_BLOCK_TYPE_COMMON` block
- power gating and clock gating
  - `nv_common_early_init`
    - `adev->cg_flags` is initialized to indicate which (sub) blocks support
      clock gating
    - `adev->pg_flags` is initialized to indicate which (sub) blocks support
      power gating
      - `amdgpu.pg_mask=0xfffffffe` masks out `AMD_PG_SUPPORT_GFX_PG`
  - `amdgpu_device_ip_set_clockgating_state` enables/disables clock gating for
    a block
  - `amdgpu_device_ip_set_powergating_state` enables/disables power gating for
    a block

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

## GFX

- `AMD_IP_BLOCK_TYPE_GFX` using `IP_VERSION(10, 3, 7)` as an example
  - `amdgpu_async_gfx_ring` is 1
  - `amdgpu_mes` is 0
- `gfx_v10_0_early_init`
  - `adev->gfx.num_gfx_rings` is set to `GFX10_NUM_GFX_RINGS_Sienna_Cichlid`
    (2)
  - `adev->gfx.num_compute_rings` is set to 8
- `gfx_v10_0_sw_init`
  - ME, micro engine (graphics)
    - `adev->gfx.me.num_me = 1`
    - `adev->gfx.me.num_pipe_per_me = 1`
    - `adev->gfx.me.num_queue_per_pipe = 1`
  - MEC, micro engine compute
    - `adev->gfx.mec.num_mec = 2`
    - `adev->gfx.mec.num_pipe_per_mec = 4`
    - `adev->gfx.mec.num_queue_per_pipe = 4`
  - `gfx_v10_0_me_init`
    - `adev->gfx.num_gfx_rings` is set to 1 by
      `amdgpu_gfx_graphics_queue_acquire`
  - `gfx_v10_0_mec_init`
    - `adev->gfx.num_compute_rings` remains 8 despite
      `amdgpu_gfx_compute_queue_acquire`
  - `gfx_v10_0_gfx_ring_init` inits `adev->gfx.gfx_ring`
    - these rings are called KGQ, kernel graphics queue
  - `gfx_v10_0_compute_ring_init` inits `adev->gfx.compute_ring`
    - these rings are called KCQ, kernel compute queue
  - `amdgpu_gfx_kiq_init_ring` inits `adev->gfx.kiq[x].ring`
    - these rings are called KIQ, kernel interface queue
- `gfx_v10_0_hw_init`
  - `gfx_v10_0_cp_resume` starts CP
    - `gfx_v10_0_kiq_resume` starts KIQ
    - `gfx_v10_0_kcq_resume` starts KCQ
      - `gfx_v10_0_cp_compute_enable` sets `mmCP_MEC_CNTL_Sienna_Cichlid`
        - this starts ME1 and ME2
      - `amdgpu_gfx_enable_kcq` submits cmds to KIQ
        - it seems some inits are done via commands submitted to KIQ
    - `gfx_v10_0_cp_async_gfx_ring_resume` starts KGQ
      - `amdgpu_gfx_enable_kgq` submits cmds to KIQ
      - `gfx_v10_0_cp_gfx_start` sets `mmCP_ME_CNTL` and submits cmds to KGQ
        - this starts CE, PFP, and ME
- EOP (End Of Pipeline) IRQ on GFX10
  - `gfx_v10_0_set_irq_funcs` initializes the IRQ functions
    - `enum amdgpu_cp_irq` enumerates EOP IRQs
      - GFX has 1 ME which has 2 PIPEs
      - COMPUTE has 2 MECs which have 4 PIPEs each
  - `gfx_v10_0_sw_init` calls `amdgpu_irq_add_id` for EOP irqs
    - the client id is `SOC15_IH_CLIENTID_GRBM_CP`
    - the source id is `GFX_10_1__SRCID__CP_EOP_INTERRUPT`
    - the src is set to `adev->irq.client[client_id].sources[src_id]`
  - `gfx_v10_0_sw_init` also calls `gfx_v10_0_gfx_ring_init` and
    `gfx_v10_0_compute_ring_init` to init the rings
  - `amdgpu_fence_driver_hw_init` calls `amdgpu_irq_get` on the irq of each
    ring
    - this calls `gfx_v10_0_set_eop_interrupt_state` to enable the interrupt
  - when a job completes, `amdgpu_irq_handler` handles the hw interrupt
    - `amdgpu_ih_process` calls `amdgpu_irq_dispatch`
    - `amdgpu_ih_decode_iv` can decode the interrupt info to
      `amdgpu_iv_entry`, which includes the client id and the source id
    - `gfx_v10_0_eop_irq` calls `amdgpu_fence_process` on the ring
    - this reads the seqno and `dma_fence_signal` completed fences

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
  - userspace
    - libdrm: `amdgpu_query_*`, `amdgpu_read_mm_registers`
    - radv also makes ioctls directly
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
  - userspace
    - libdrm: `amdgpu_cs_ctx_*`, `amdgpu_cs_query_reset_state*`
    - `amdgpu_cs_query_reset_state*` only used by radeonsi
  - radv creates a ctx for each `VkQueue`
  - radv uses `AMDGPU_CTX_OP_QUERY_STATE2` for gpu hang check
    - `AMDGPU_CTX_QUERY2_FLAGS_RESET` and `AMDGPU_CTX_QUERY2_FLAGS_GUILTY` are
      sticky and the context should be re-created
  - radv uses `AMDGPU_CTX_OP_SET_STABLE_PSTATE` to force peak performance in
    `vkAcquireProfilingLockKHR`
- `DRM_IOCTL_AMDGPU_VM` reserves the vmid
  - userspace
    - libdrm: `amdgpu_vm_{reserve,unreserve}_vmid`
    - it is used for shader debugging
- `DRM_IOCTL_AMDGPU_GEM_CREATE` creates a gem bo
  - userspace
    - libdrm: `amdgpu_bo_alloc`
- `DRM_IOCTL_AMDGPU_GEM_USERPTR` creates a gem bo from userptr
  - userspace
    - libdrm: `amdgpu_create_bo_from_user_mem`
- `DRM_IOCTL_AMDGPU_GEM_MMAP` returns the magic mmap offset for cpu mapping
  - userspace
    - libdrm: `amdgpu_bo_cpu_map`
    - radeonsi uses libdrm wrapper and radv uses ioctl directly
- `DRM_IOCTL_AMDGPU_GEM_METADATA` sets/gets the bo metadata
  - userspace
    - libdrm: `amdgpu_bo_set_metadata`, `amdgpu_bo_query_info`
    - it is for bo export/import
  - `drm_amdgpu_gem_metadata` payload has
    - `__u64 flags` is reserved
    - `__u64 tiling_info` is the tiling flags
      - DCE looks at this, likely because the userspace does not handle it
    - `__u32 data[64]` is completely opaque
- `DRM_IOCTL_AMDGPU_GEM_OP` queries/updates gem bo info
  - userspace
    - libdrm: `amdgpu_bo_query_info`
    - for bo import
    - no `AMDGPU_GEM_OP_SET_PLACEMENT` support
- `DRM_IOCTL_AMDGPU_GEM_VA` maps/unmaps/replaces gem bo va
  - userspace
    - libdrm: `amdgpu_bo_va_op_raw`, `amdgpu_bo_va_op`
- `DRM_IOCTL_AMDGPU_CS` submits a job
  - userspace
    - libdrm: `amdgpu_cs_submit_raw*`, `amdgpu_cs_submit`
  - `drm_amdgpu_cs_out::handle` is the seqno of the job
- `DRM_IOCTL_AMDGPU_WAIT_CS` waits for a job
  - userspace
    - libdrm: `amdgpu_cs_query_fence_status`
    - only used by radv; radeonsi uses `amdgpu_bo_wait_for_idle`
- somewhat legacy
  - `DRM_IOCTL_AMDGPU_SCHED` changes the priority of any context
    - userspace
      - libdrm: `amdgpu_cs_ctx_override_priority`
      - no user
    - master node only
  - `DRM_IOCTL_AMDGPU_BO_LIST` creates/destroys/updates a bo list
    - userspace
      - libdrm: `amdgpu_bo_list_*`
      - no user, deprecated by `AMDGPU_CHUNK_ID_BO_HANDLES`
  - `DRM_IOCTL_AMDGPU_GEM_WAIT_IDLE` waits on the implicit fence of a bo
    - userspace
      - libdrm: `amdgpu_bo_wait_for_idle`
      - only used by radeonsi
  - `DRM_IOCTL_AMDGPU_FENCE_TO_HANDLE` converts a `drm_amdgpu_fence` to a
    syncobj, syncobj fd, or sync file fd
    - userspace
      - libdrm: `amdgpu_cs_fence_to_handle`
      - no user since radeonsi moved to syncobj
  - `DRM_IOCTL_AMDGPU_WAIT_FENCES` waits an array of `drm_amdgpu_fence`s
    - userspace
      - libdrm: `amdgpu_cs_wait_fences`
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
      - when `AMDGPU_IB_FLAG_CE` is set, submits to CE rather than ME
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

## `amdgpu_sync`

- `amdgpu_cs_ioctl` usage
  - `amdgpu_cs_parser_init` calls `amdgpu_sync_create` to initialize `p->sync`
    - an `amdgpu_sync` is used in several other places, to manage fences
  - `amdgpu_cs_p2_dependencies` calls `amdgpu_sync_fence` to add dep fences
  - `amdgpu_cs_p2_syncobj_in` and `amdgpu_cs_p2_syncobj_timeline_wait` calls
    call `amdgpu_sync_fence` to add syncobj fences
  - `amdgpu_cs_vm_handling` calls `amdgpu_sync_fence` to add pt update fence
  - `amdgpu_cs_sync_rings` calls `amdgpu_sync_resv` to add bo resvs
    - it also calls `amdgpu_sync_push_to_job` to add managed fences to as job
      dependencies
    - finally, it moves fences from `p->sync` to
      `p->gang_leader->explicit_sync`
  - `amdgpu_cs_parser_fini` calls `amdgpu_sync_free` to clean up `p->sync`
- `amdgpu_sync_create` initializes an `amdgpu_sync`
  - `sync->fences` is a hash table from fence contexts to `amdgpu_sync_entry`
- `amdgpu_sync_fence` adds a fence to `sync->fences`
  - `amdgpu_sync_add_later` checks if a prior fence of the same fence context
    exists and keeps the later one
- `amdgpu_sync_resv` adds a bo resv to `sync`
  - it loops through all fences (`DMA_RESV_USAGE_BOOKKEEP`) associated with
    the resv
  - if a fence passes `amdgpu_sync_test_fence`, it is added using the regular
    `amdgpu_sync_fence`
- `amdgpu_sync_test_fence` is the interesting part
  - `amdgpu_sync_resv` calls `amdgpu_sync_test_fence` to test if an implict
    fence should be added to `amdgpu_sync`
    - in cs submit, we call `amdgpu_sync_resv` with `fpriv->vm` as the owner
  - if an implicit fence is a `drm_sched_fence`, it has a meaningful owner
    - in cs submit, we call `amdgpu_job_alloc` with `fpriv->vm` as the owner
  - if an implicit fence has a meaningful owner, we test it against `mode`
    - `AMDGPU_SYNC_ALWAYS` returns true
    - `AMDGPU_SYNC_EXPLICIT` returns false
    - `AMDGPU_SYNC_NE_OWNER` returns true if the fence has a different owner
    - `AMDGPU_SYNC_EQ_OWNER` returns true if the fence has the samae owner
    - in cs submit, `mode` is either
      - `AMDGPU_SYNC_NE_OWNER`, the default, or
      - `AMDGPU_SYNC_EXPLICIT` if the bo was created with
        `AMDGPU_GEM_CREATE_EXPLICIT_SYNC`
- IOW, when a cs submit calls `amdgpu_sync_resv`,
  - if an implicit fence is from another driver, it is honored
  - if an implicit fence is from another amdgpu cs submit,
    - if the bo was created with `AMDGPU_GEM_CREATE_EXPLICIT_SYNC`, `mode` is
      `AMDGPU_SYNC_EXPLICIT` and the implicit fence is ignored
    - otherwise, `mode` is `AMDGPU_SYNC_NE_OWNER` and the implicit fence is
      ignored unless it is from a different VM
  - due to how `amdgpu_device_initialize` in userspace works, all
    `libdrm_amdgpu` users in the same process share the same VM and there is
    no implicit fencing between two `libdrm_amdgpu` users in the same process

## GEM BOs

- libdrm `amdgpu_bo_alloc` or ioctl `DRM_IOCTL_AMDGPU_GEM_CREATE` creates a
  gem bo
- `struct drm_amdgpu_gem_create`
  - `bo_size`
  - `alignment`
  - `domains` is the initial/preferred domains
    - `AMDGPU_GEM_DOMAIN_CPU` is system memory that is not gpu-accessible
    - `AMDGPU_GEM_DOMAIN_GTT` is system memory that is gpu-accessible
    - `AMDGPU_GEM_DOMAIN_VRAM` is video memory (or carved-out on apu)
    - `AMDGPU_GEM_DOMAIN_GDS` is on-chip global data storage
    - `AMDGPU_GEM_DOMAIN_GWS` is global wave sync
    - `AMDGPU_GEM_DOMAIN_OA` is ordered append
  - `domain_flags`
    - `AMDGPU_GEM_CREATE_CPU_ACCESS_REQUIRED` is for vram, indicating cpu-accessible
    - `AMDGPU_GEM_CREATE_NO_CPU_ACCESS` is for vram, indicating no cpu access
    - `AMDGPU_GEM_CREATE_CPU_GTT_USWC` is for gtt, indicating WC
      - it is ignored when `amdgpu_bo_support_uswc` returns false
      - it affects `amdgpu_ttm_tt_create`, to choose between
        `ttm_write_combined` and `ttm_cached`
    - `AMDGPU_GEM_CREATE_VRAM_CLEARED` is for vram
    - `AMDGPU_GEM_CREATE_VM_ALWAYS_VALID`
    - `AMDGPU_GEM_CREATE_EXPLICIT_SYNC` indicates explicit sync
      - it is being deprecated by
        <https://lists.freedesktop.org/archives/amd-gfx/2024-August/112292.html>
    - `AMDGPU_GEM_CREATE_ENCRYPTED` indicates protected contents
    - `AMDGPU_GEM_CREATE_DISCARDABLE` indicates the contents can be discarded
      on memory pressure
- `amdgpu_gem_create_ioctl` calls `amdgpu_bo_create` to allocate
  - both `bo->preferred_domains` and `bo->allowed_domains` are set to the
    domains specified by the ioctl
  - `amdgpu_bo_placement_from_domain` sets bo placements based on domains
    - `AMDGPU_GEM_DOMAIN_VRAM` means `TTM_PL_VRAM`
      - `AMDGPU_GEM_CREATE_CPU_ACCESS_REQUIRED` requires an offset before
        `visible_vram_size`
    - `AMDGPU_GEM_DOMAIN_GTT` means `TTM_PL_TT`
    - `AMDGPU_GEM_DOMAIN_CPU` means `TTM_PL_SYSTEM`
  - `ttm_bo_init_reserved` initializes the tbo struct
  - if `AMDGPU_GEM_CREATE_VRAM_CLEARED` and `TTM_PL_VRAM`,
    `amdgpu_fill_buffer` clears the buffer to 0
- scanout
  - `amdgpu_mode_dumb_create` creates a dumb bo
    - `amdgpu_display_supported_domains` returns the allowed domains
      - the hw can only scan out from vram, or gtt that is USWC
      - this returns `AMDGPU_GEM_DOMAIN_VRAM` and condtionally
        `AMDGPU_GEM_DOMAIN_GTT`
    - `amdgpu_bo_get_preferred_domain` returns the preferred domains based on
      allowed domains
      - it simply returns the allowed domains after stoney
  - `amdgpu_dma_buf_begin_cpu_access` moves the bo to gtt if needed and
    possible
    - if read access, attemps to move to gtt
    - possible if gtt is an allowed domain, USWC is set, etc.
  - `amdgpu_dm_plane_helper_prepare_fb` calls `amdgpu_bo_pin` to pin the bo
    - if overlay, it pins to `amdgpu_display_supported_domains`
    - if cursor, it pins to `AMDGPU_GEM_DOMAIN_VRAM`
    - it is fine if the domain is not in the intial/preferred domains
      - but note that vram is carveout on apu and can be very small
      - it is not possible to fit too many bos in vram on apu, and we want
        `AMDGPU_GEM_CREATE_CPU_GTT_USWC` such that the bos can be in gtt

## Modifiers

- mesa `ac_get_supported_modifiers` returns all supported modifiers in
  preference order
- gfx9
  - `AMD_FMT_MOD_SET(TILE_VERSION, AMD_FMT_MOD_TILE_VER_GFX9)`
  - dcc
    - `common_dcc`
      - `AMD_FMT_MOD_SET(DCC, 1)`
      - `AMD_FMT_MOD_SET(DCC_INDEPENDENT_64B, 1)`
      - `AMD_FMT_MOD_SET(DCC_MAX_COMPRESSED_BLOCK, AMD_FMT_MOD_DCC_BLOCK_64B)`
      - `AMD_FMT_MOD_SET(DCC_CONSTANT_ENCODE, info->has_dcc_constant_encode)`
      - `AMD_FMT_MOD_SET(PIPE_XOR_BITS, pipe_xor_bits)`
      - `AMD_FMT_MOD_SET(BANK_XOR_BITS, bank_xor_bits)`
    - `AMD_FMT_MOD_SET(TILE, AMD_FMT_MOD_TILE_GFX9_64K_D_X)`
      - `AMD_FMT_MOD_SET(DCC_PIPE_ALIGN, 1)`
      - with `PIPE` and `RB` are from `gb_addr_config`
    - `AMD_FMT_MOD_SET(TILE, AMD_FMT_MOD_TILE_GFX9_64K_S_X)`
      - `AMD_FMT_MOD_SET(DCC_PIPE_ALIGN, 1)`
      - with `PIPE` and `RB` are from `gb_addr_config`
      - if 32-bit
        - if 1 RB, another variant
        - yet another variant with `DCC_RETILE`
  - no dcc
    - `AMD_FMT_MOD_SET(TILE, AMD_FMT_MOD_TILE_GFX9_64K_D_X)`
      - with `PIPE_XOR_BITS` and `BANK_XOR_BITS` from `gb_addr_config`
    - `AMD_FMT_MOD_SET(TILE, AMD_FMT_MOD_TILE_GFX9_64K_S_X)`
      - with `PIPE_XOR_BITS` and `BANK_XOR_BITS` from `gb_addr_config`
    - `AMD_FMT_MOD_SET(TILE, AMD_FMT_MOD_TILE_GFX9_64K_D)`
    - `AMD_FMT_MOD_SET(TILE, AMD_FMT_MOD_TILE_GFX9_64K_S)`
- let's try to decipher gfx9 modifiers
  - there are two basic macro tiles
    - `AMD_FMT_MOD_TILE_GFX9_64K_S`, each macro tile is 64KB with a
      standard tiling
    - `AMD_FMT_MOD_TILE_GFX9_64K_D`, each macro tile is 64KB with a
      displayable tiling
  - the two macro tiles have variants that respect XOR bits
    - `AMD_FMT_MOD_TILE_GFX9_64K_S_X` and `AMD_FMT_MOD_TILE_GFX9_64K_D_X`
    - it is called "tile swizzle" in mesa
    - when doing MRTs, mesa sets tile swizzles to different values in the
      surface descriptors for the RTs
    - when the shader accesses the MRTs, tile swizzles are used to perform
      XORs on the addrs, such that the access hits different pipes/banks and
      maximize utilization
  - DCC is a separate surface
    - the main surface must be in `_X` varants of macro tiles
    - `DCC_CONSTANT_ENCODE` means the fast clear color may be stored in the
      DCC
      - on older hw, the fast clear color is ignored and a fast clear
        elimination (FCE) pass must be performed
    - `DCC_INDEPENDENT_64B`, `DCC_INDEPENDENT_128B`, and
      `DCC_MAX_COMPRESSED_BLOCK` seem to affect how DCC compresses
      - in mesa `gfx9_compute_surface`, it picks a combo that is optimal for
        L2 cache
      - if the image needs to be displayable, the settings must meet display
        hw limitations as well
    - `DCC_PIPE_ALIGN` seems to affect alignment
      - the CB block requires `RB_ALIGNED=1`, unless there is only 1 RB
      - the CB block prefers `PIPE_ALIGNED=1`, otherwise it needs to flush L2
        after rending
        - `PIPE_ALIGNED` actually means `L2CACHE_ALIGNED`
      - the diplsya hw requires `RB_ALIGNED=0` and `PIPE_ALIGNED=0`
      - if there is only 1 RB, it makes sense to `RB_ALIGNED=0` and
        `PIPE_ALIGNED=0`
      - otherwise, uses `RB_ALIGNED=1` and `PIPE_ALIGNED=1`, and retile
    - `DCC_RETILE` means the CB and the display use different DCC surfaces
      - this is required when the CB and the display have different reqs
      - mesa `radv_retile_dcc` retiles the DCC for display
    - `PIPE` and `RB` depend on `gb_addr_config` and affect DCC
- gfx8
  - there is no modifier support yet
  - `gfx_v8_0_constants_init` writes golden value to `mmGB_ADDR_CONFIG`
  - `gfx_v8_0_tiling_mode_table_init` writes `mmGB_TILE_MODE*` and
    `mmGB_MACROTILE_MODE*`

## BO metadata

- before modifiers, `DRM_IOCTL_AMDGPU_GEM_METADATA` and
  `amdgpu_gem_metadata_ioctl` are used for bo metadata
- there is a 64-bit `tiling_info` used by both kernel and user spaces
  - `amdgpu_bo_set_tiling_flags` saves the 64-bit tiling info to
    `bo->tiling_flags`
  - `AMDGPU_TILING_*` defines the bits
    - gfx8 has
      - `ARRAY_MODE`
      - `PIPE_CONFIG`
      - `TILE_SPLIT`
      - `MICRO_TILE_MODE`
      - `BANK_WIDTH`
      - `BANK_HEIGHT`
      - `MACRO_TILE_ASPECT`
      - `NUM_BANKS`
    - gfx9+ has
      - `SWIZZLE_MODE`
      - `DCC_OFFSET_256B` is the offset of the dcc metadata in 256-bytes
      - `DCC_PITCH_MAX` is the pitch of the dcc metadata
      - `DCC_INDEPENDENT_64B`
      - `DCC_INDEPENDENT_128B`
      - `SCANOUT`
  - gfx9+ has modifiers and no longer uses the tiling info internally
    - when the userspace uses tiling info, `convert_tiling_flags_to_modifier`
      converts the tiling info to a modifier and the rest of the driver deals
      with modifiers
  - `amdgpu_dm_plane_fill_gfx8_tiling_info_from_flags` parses the tiling info
    to `dc_tiling_info` on gfx8
- there is a 256-byte blob used by userspace
  - the kernel only needs the 64-bit tiling info, for display
  - this 256-byte blob is completely opaque to the kernel
