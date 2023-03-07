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

## FMASK

- FMASK is optional and is used for MSAA compression up to GFX10
  - it is removed from GFX11+
- <https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_AMD_shader_fragment_mask.html>
  - each pixel has N samples
  - when two samples share the same value, the value is stored only once
  - a fragment mask (FMASK) is used to map samples to stored values
  - to fetch sample 0,
    - `fragmentFetchAMD(tex, coord, (fragmentMaskFetchAMD(tex, coord) & 0x4)`
- `Addr2ComputeFmaskInfo` computes FMASK layout
  - it is clear that FMASK is just another texture
    - it can be tiled and compressed
  - `pIn->numSamples` is number of samples
  - `pIn->numFrags` is number of samples to store
- `V_028808_CB_FMASK_DECOMPRESS` decompresses FMASK
  - I guess CMASK stores metadata about FMASK to compress FMASK
  - this op resolves CMASK to FMASK, such that `fragmentMaskFetchAMD` works
    - this also implies `V_028808_CB_ELIMINATE_FAST_CLEAR`
  - this op is unnecessary when CMASK is TC-compatible
- I guess `V_028808_CB_DECOMPRESS` "expands" FMASK
  - it is not used by drivers
  - instead, drivers can "expand" FMASK using a compute shader which I guess
    does the same thing
  - what this does is to resolve FMASK to the main MSAA surface

## CMASK

- CMASK is used for fast clear and FMASK up to GFX10
  - it is only used for FMASK on GFX10
  - it is removed from GFX11+
- `Addr2ComputeCmaskInfo` computes CMASK layout
- each tile has 4 bits in CMASK
  - if there is also DCC,
    - the higher 2 bits must be 1's, to indicate no cmask fast clear
      - there can still be dcc fast clear
    - the lower 2 bits indicate compression level
      - 0 means full compression
      - 3 means no compression
  - if there is no DCC,
    - if the higher 2 bits are 1's, it indicates no cmask fast clear
      - the lower 2 bits means
        - 0 is unused?
        - 1 means 2 samples
        - 2 means 4 samples
        - 3 means 8 samples or 1 sample
    - otherwise, it indicates cmask fast clear

## DCC

- DCC is used for fast clear and compression
  - it replaces FMASK and CMASK completely on GFX11+
  - FMASK and CMASK are internal to drivers
  - DCC, on the other hand, is shared externally
- `Addr2ComputeDccInfo` computes DCC layout

## Terms

- R600 pipeline
  - CP & GRBM
    - command processor
  - VGT
    - vertex grouper / tesselator
  - PA
    - primitive assembly
  - SC
    - scan converter
  - SPI
    - shader pipe interpolator
  - shader
    - TC, texture cache
    - VC, vertex cache
    - SMX, shader memory exchange
  - SX
  - DB
    - depth buffer
  - CB
    - color buffer
- image compression
  - MSAA can be compressed or uncompressed
    - when compressed, it requires CMASK and FMASK
  - fast clear requires CMASK
    - FMASK is not required
  - HTILE
    - a separate surface that holds the metadata for compression and
      hierarchical optimization
    - for hiz?
  - CMASK
    - for fast clear and for msaa compression
  - FMASK
    - for msaa compression
  - DCC
    - color compression
- `R_028808_CB_COLOR_CONTROL`
  - `V_028808_CB_DISABLE` disables color write
  - `V_028808_CB_NORMAL` enables color write
  - `V_028808_CB_ELIMINATE_FAST_CLEAR` eliminates fast clear
    - FCE, fast clear eliminate
  - GFX10-
    - `V_028808_CB_RESOLVE` resolves MSAA
    - `V_028808_CB_DECOMPRESS` is unused
    - `V_028808_CB_FMASK_DECOMPRESS` decompresses FMASK
    - `V_028808_CB_DCC_DECOMPRESS_GFX8` decompresses DCC
  - GFX11+
    - `V_028808_CB_DCC_DECOMPRESS_GFX11` decomresses DCC
