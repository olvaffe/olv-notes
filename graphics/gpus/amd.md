AMD
===

## History

- ATI Rage released in 1996 to 1999
- R100 released in 2000
  - Direct3D 7.0
  - `radeon_dri.so`
- R200 released in 2001
  - Direct3D 8.1 w/ shader model 1.4
  - `r200_dri.so`
- R300 released in 2002
  - Direct3D 9.0 w/ shader model 2.0
  - `r300_dri.so`
- R400 released in 2004
  - Direct3D 9.0b w/ shader model 2.0b
  - `r300_dri.so`
- R500 released in 2005
  - Direct3D 9.0c w/ shader model 3.0
  - `r300_dri.so`
- R600 released in 2007
  - Direct3D 10.1 w/ shader model 4.1
  - `r600_dri.so`
- R700 released in 2008
  - Direct3D 10.1 w/ shader model 4.1
  - `r600_dri.so`
- Evergreen released in 2009
  - Direct3D 11.0 w/ `11_0` feature level
  - `r600_dri.so` Evergreen
- Northern Islands released in 2010
  - Direct3D 11.0 w/ `11_0` feature level
  - `r600_dri.so` Cayman
- Southern Islands released in 2012
  - Direct3D 12.0 w/ `11_1` feature level
  - `radeonsi_dri.so` GFX6
  - GCN1
- Sea Islands released in 2013
  - Direct3D 12.0 w/ `11_1` feature level
  - `radeonsi_dri.so` GFX7
  - GCN2
- Volcanic Islands released in 2014
  - Direct3D 12.0 w/ `12_0` feature level
  - `radeonsi_dri.so` GFX8
  - GCN3
- Polaris released in 2016
  - Direct3D 12.0 w/ `12_0` feature level
  - `radeonsi_dri.so` GFX8
  - GCN4
- Vega released in 2017
  - Direct3D 12.0 w/ `12_1` feature level
  - `radeonsi_dri.so` GFX9
  - GCN5
- Navi 1x released in 2019
  - Direct3D 12.0 w/ `12_1` feature level
  - `radeonsi_dri.so` GFX10
  - RDNA
- Navi 2x released in 2020
  - Direct3D 12.0 w/ `12_2` feature level
  - `radeonsi_dri.so` `GFX10_3`
  - RDNA 2
- Navi 3x released in 2022
  - RDNA 3
  - 5nm
  - chiplet

## RDNA

- AMD Navi Radeon RX 5700 XT
- $399
- 2 Shader Engines, each has
  - 2 64-bit Memory Controllers, each has
    - 4 256KB slices of L2
  - 2 Shader Arrays, each has
    - 1 L1
    - 1 Primitive Unit
      - Primitive Assembly
      - Tessellation
    - 1 Rasterizer
      - Rasterization
    - 4 Render Backends
      - Depth/Stencil/Alpha Test
      - Blend
    - 5 Dual Compute Units
- a Dual Compute Unit has
  - 1 32KB L0
  - 1 128KB Local Data Share
  - 2 LD/ST/Tex Units
  - 4 SIMDs, each has
    - ? Schedulers
    - 32 ALUs
      - together can execute one instruction each cycle for wave32
    - 1 128KB Vector Register File
      - 1024 32-bit registers for each ALU
    - 1 scalar ALU
    - 1 10KB Scalar Register File
- the primitive unit can cull 2 primitves and output 1 primitive to
  rasterizer per clock
  - cull rate is `engine count * array count * 2 * mhz = 2*2*2*1905 = 15240`
  - output rate is 7620 mtris/s
- the rasterizer can output up to 16 pixels per clock
  - output rate is `engine count * array count * 16 * mhz = 2*2*16*1905 =
    121920` mpixels/s
- flops are
  - `engine count * array count * cu count * simd count * simd width * fma =
     2 * 2 * 5 * 4 * 32 * 2 = 5120` flops/cycle
  - 9.753 tflops/s

## GCN

- Micro Engine / Micro Engine Compute, ME / MEC
  - there is a ME, aka graphics command processor, aka CP
    - only one queue?  thus the kernel assigns each process a software queue,
      and copies commands from the SW queue to the single HW queue, to avoid
      one process taking over the queue
    - the queue supports DRAW and COMPUTE commands
    - in the old days, ME had one graphics queue and two compute queues
  - there are one or two (or more) MECs
    - each MEC has 4 threads, aka pipes, aka ACEs (asyn compute engine)
    - each pipe has 8 compute queues, aka compute rings
    - e.g., the kernel exposes only some of the queues to the userspace
      - hw with 1 MEC: all 8 queues in the first pipe
      - hw with 2 MECs: first 2 queues of each of the 4 pipes in the first MEC
    - the queues support only COMPUTE commands
- Hardware Scheduler, HWS
  - dispatch tasks from ME/MECs to CUs?
- Shader Engine, SE
  - a SE consists of 1 geometry processor (1 primitive/cycle), 1 rasterizer, 1
    to 16 CUs, 1 to 4 Render Back Ends (RBEs); I think a SE is one
    fixed-function graphics pipeline.
  - a GPU normally has 1 to 4 SEs; each SE has `max_sh_per_se * max_cu_per_sh`
    CUs; on higher-end GPUs, the total number of CUs is in the 64 ballpark
- Compute Unit, CU
  - a CU consists of a CU scheduler, a branch&message unit, 4 SIMD-VUs, 4
    64KiB register arrays, and 4 Texture Filter Units.
  - CU scheduler is different from HWS
    - it groups threads into wavefronts and assigns wavefronts to SIMD-VUs
- SIMD Vector Unit, SIMD-VU
  - 256 vector registers, each consists of 64 floats (64KiB in total)
  - a wavefront consists of 64 threads, with thread N using channel N of the
    registers
  - can execute up to 10 wavefronts at the same time, depending on how many
    registers a thread needs
    - e.g., when a fragment shader needs 32 registers, there can be 8
      wavefronts processing 512 fragments at the same time
  - has 16-lane ALU; thus it takes 4 cycles to execute one wavefront

## Kernel Driver

- for each `drm_device`, `amdgpu_driver_load_kms` is called
  - depending on the HW generation, it adds a bunch of IP blocks with
    `amdgpu_device_ip_block_add`
  - each IP block goes through `early_init`, `sw_init`, `hw_init`, and
    `late_init`
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
- `DRM_IOCTL_AMDGPU_INFO` with `AMDGPU_INFO_MEMORY` returns the memory
  information
  - it reports the sizes of three heaps: `vram`, `cpu_accessible_vram`, and
    `gtt`
  - vram size is `adev->gmc.real_vram_size`
  - `cpu_accessible_vram` size is `adev->gmc.visible_vram_size`
  - `gtt` size is `adev->mman.bdev.man[TTM_PL_TT].size`
  - there is also `AMDGPU_INFO_VRAM_GTT` which is deprecated
- `DRM_IOCTL_AMDGPU_GEM_CREATE`
  - `amdgpu_bo_do_create`
  - all flags are trasnalted to a `ttm_placement` and a ttm bo is created
- `DRM_IOCTL_AMDGPU_GEM_VA`
  - the userspace finds a 48-bit VA for the bo and call this ioctl with
    `AMDGPU_VA_OP_MAP`
  - translate userspace flags to `va_flags`: read/write/exec, NC/WC/CC/UC
  - assocaite a `amdgpu_bo_va_mapping` with the BO
  - i guess at some point, the BO is mapped into the VM
- `DRM_IOCTL_AMDGPU_GEM_MMAP` sets up a mmap offset and mmap is handled by
  `ttm_bo_mmap`
  - `ttm_bo_mmap` installs `ttm_bo_vm_fault` as the fault handler
  - the fault handler calls `->fault_reserve_notify` to move the BO to a
    visible place: no-op if GTT, move to BAR 0 if VRAM
  - the fault handler calls `->io_mem_reserve`
  - if GTT, get the pfn of the page; if VRAM, `->ttm_bo_io_mem_pfn` to get the
    pfn of an address in BAR 0
  - pte is set up as requested: UC, WC or WB
    - VRAM is either UC or WC
    - GTT can also be WB.  When WB, `AMDGPU_PTE_SNOOPED` is also set in GPU VM

## Heaps and Memory Mapping

- radv heaps and memory types are
  - heap 0: vram minus cpu-accessible-vram
  - heap 1: cpu-accessible-vram
  - heap 2: gtt (`radeon_info` calls it gart, but it is gtt)
  - type 0 is called VRAM: use heap 0
  - type 1 is called `GTT_WRITE_COMBINE`: use heap 2 w/ coherency w/o cache
  - type 2 is called `VRAM_CPU_ACCESS`: use heap 1 w/ coherency w/o cache
  - type 3 is called `GTT_CACHED`: use heap 2 w/ coherency w/ cache
- `vkAllocateMemory`
  - userspace is responsbile for managing the VM of GPU for itself
  - it find a VA in the VM
  - it allocate a BO with `DRM_IOCTL_AMDGPU_GEM_CREATE`
  - it maps the BO into the VM with `DRM_IOCTL_AMDGPU_GEM_VA`
- External Memory Import
  - getting the BO handle
    - for prime, it is the standard `DRM_IOCTL_PRIME_FD_TO_HANDLE`
    - for userptr, it is `DRM_IOCTL_AMDGPU_GEM_USERPTR`
  - it is then followed by finding a VA for the imported BO and map it into
    the VM
- `vkMapMemory`
  - if userptr, return directly
  - otherse, `DRM_IOCTL_AMDGPU_GEM_MMAP` followed by an mmap

## Command Processor

- Relation to Adreno
  - ATI was founded in 1993 and acquired by AMD in 2006
  - ATI introduced Imageon in 2002 which was acquired by Qualcomm in 2009
  - Imageon Z430 was incorporated into the Qualcomm MSM7x27 and QSD8x50 series
    of processors, being rebranded as the Adreno 200
- PM4 Packets
  - Type-0 should be avoided in favor of Type-3
    - it writes consecutive registers
  - Type-2 is filler for trailing space
  - Type-3 has an opcode
- SI
  - there are two CP engines on SI
    - DE, drawing engine, which was previously known as ME, micro engine
    - CE, constant engine
  - `ME_INITIALIZE` initializes ME (known as DE since SI)
  - `PFP_SYNC_ME` stalls PFP until ME catches up
  - `WAIT_REG_MEM` can be executed on PFP or ME
    - it stalls until reg/mem meets the condition
  - `COPY_DATA` can be executed on ME or CE
