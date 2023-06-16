AMD
===

## GPU uArchs

- fixed pipeline
  - ATI Rage released in 1996 to 1999
- separted VS and FS
  - R100 released in 2000
    - Direct3D 7.0
    - `radeon_dri.so`
  - R200 released in 2001
    - Direct3D 8.1 w/ shader model 1.4
    - `r200_dri.so`
  - R300 released in 2002
    - Direct3D 9.0 w/ shader model 2.0
    - `r300_dri.so`
    - `R300`
  - R400 released in 2004
    - Direct3D 9.0b w/ shader model 2.0b
    - `r300_dri.so`
    - `R400`
  - R500 released in 2005
    - Direct3D 9.0c w/ shader model 3.0
    - `r300_dri.so`
    - `R500`
- TeraScale
  - `r600_dri.so`
  - R600 released in 2007
    - Direct3D 10.1 w/ shader model 4.1
    - `R600`
  - R700 released in 2008
    - Direct3D 10.1 w/ shader model 4.1
    - `R700`
  - Evergreen released in 2009
    - Direct3D 11.0 w/ `11_0` feature level
    - `EVERGREEN`
  - Northern Islands released in 2010
    - Direct3D 11.0 w/ `11_0` feature level
    - `CAYMAN`
- GCN
  - `radeonsi_dri.so`
  - Southern Islands released in 2012
    - Direct3D 12.0 w/ `11_1` feature level
    - GCN1
    - `GFX6`
  - Sea Islands released in 2013
    - Direct3D 12.0 w/ `11_1` feature level
    - GCN2
    - `GFX7`
  - Volcanic Islands released in 2014
    - Direct3D 12.0 w/ `12_0` feature level
    - GCN3
    - `GFX8`
  - Polaris released in 2016
    - Direct3D 12.0 w/ `12_0` feature level
    - GCN4
    - `GFX8`
  - Vega released in 2017
    - Direct3D 12.0 w/ `12_1` feature level
    - GCN5
    - `GFX9`
- RDNA
  - `radeonsi_dri.so`
  - Navi 1x released in 2019
    - Direct3D 12.0 w/ `12_1` feature level
    - RDNA
    - `GFX10`
  - Navi 2x released in 2020
    - Direct3D 12.0 w/ `12_2` feature level
    - RDNA 2
    - `GFX10_3`
  - Navi 3x released in 2022
    - RDNA 3
    - 5nm
    - chiplet
    - `GFX11`

## Naming

- e.g., `Radeon RX 7900 XTX`
- `Radeon RX`
  - `RX` is in the name since 2017
    - it stands for `Radeon Experience`
  - ancient
    - `Rage` before 2000
    - `7000`, `8000`, `9000` in early 2000s
    - `X200`, `X300`, ... around 2005
    - `HD 2000` to `HD 8000` until early 2013
    - `200`, `300`, `400`, `500` until 2017
- generations
  - the first number is generation
  - `5000` is RDNA1
  - `6000` is RDNA2
  - `7000` is RDNA3
- performances
  - the last 3 numbers are performance levels
  - `6600 < 6650 < 6700`
  - `500` is entry-level, $2xx
  - `600` is mid-range, $3xx
  - `700` and `800` are high-end, $5xx
  - `900` is enthusiast, $9xx
- suffices
  - `XT` is faster
    - it stands for `Extreme`
    - `6700 < 6700 XT < 6800`
  - `XTX` is even faster

## Documentations

- Radeon R3xx 3D Register Reference Guide
  - <https://www.amd.com/system/files/TechDocs/R3xx_3D_Registers.pdf>
- Radeon R6xx/R7xx Acceleration and 3D Register Reference Guide
  - <https://www.amd.com/system/files/TechDocs/R6xx_R7xx_3D.pdf>
  - <https://www.amd.com/system/files/TechDocs/R6xx_3D_Registers.pdf>
- Radeon Evergreen/Northern Islands Acceleration and 3D Register Reference
  Guide
  - <https://www.amd.com/system/files/TechDocs/evergreen_cayman_programming_guide.pdf>
  - <https://www.amd.com/system/files/TechDocs/evergreen_3D_registers_v2.pdf>
- Radeon Southern Islands Acceleration and 3D/Compute Register Reference Guide
  - <https://www.amd.com/system/files/TechDocs/si_programming_guide_v2.pdf>
  - <https://www.amd.com/system/files/TechDocs/SI_3D_registers.pdf>
- Radeon Sea Islands 3D/Compute Register Reference Guide
  - <https://www.x.org/docs/AMD/old/CIK_3D_registers_v2.pdf>

## RDNA 3

- Radeon RX 7900 XTX
- $999
- there is 1 GCD and 6 MCDs
  - GCD, Graphics Chiplet Die, is 5nm
  - each MCD, Memory Chiplet Die, is 6nm and provides 16MB of infinity cache
  - GCD and MCDs are interconnected with infinity fabric
- inside the GCD,
  - 1 PCIe Gen4 block
  - 24 256KB L2 for a total of 6MB
  - 1 fixed-function block
  - 6 programmable shader engines
  - 1 multimedia engine
  - 1 display engine
- Command Processor probably lives in the PCIe Gen4 block
  - it parses the command stream and dispatches commands
  - media comamnds are dispatched to multimedia engine
  - graphics and compute commands are dispatched to the fixed-function block
- L2 is shared by all blocks
- inside the fixed-function block,
  - 1 graphics command processor that supports graphics and compute
  - 4 ACEs (async compute engines) that support compute only
  - 1 DMA that supports 2D
  - 1 geometry processor that does geometry processing
  - 1 HWS for hardware scheduling?
- inside each shader engine,
  - 1 prim unit
  - 1 rasterizer
  - 2 shader arrays, I guess, and each shader array has
    - 1 128KB L1s
    - 2 RB+
    - 4 WCPs/DCUs (Workgroup Processor or Dual Compute Unit)
- inside each WCP/DCU,
  - 1 32KB instruction cache
  - 1 16KB scalar cache
  - 1 shared memory
  - 2 CUs
- inside each CU,
  - 1 32KB L0
  - 1 texture filter
  - 1 ld/st/tex addr
  - 1 ray accelerator
  - 1 scheduler
  - ? scalar ALUs and scalar GPRs
  - ? vecor ALUs and vector GPRs
- there is a total of `1 (gcd) * 6 (se) * 2 (sa) * 4 (wcp) * 2 (cu) = 96` CUs

## RDNA

- <https://www.amd.com/system/files/documents/rdna-whitepaper.pdf>
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
- all commands used by radv
  - `PKT3_ACQUIRE_MEM` is for semaphore?
  - `PKT3_ATOMIC_MEM` performs an atomic op to addr
  - `PKT3_CLEAR_STATE` is for dev init
  - `PKT3_COND_EXEC` executes the following N dwords only if addr is non-zero
  - `PKT3_CONTEXT_CONTROL` is for dev init
  - `PKT3_COPY_DATA` copies data from src to dst
    - src can be imm, reg, addr, etc.
    - dst can be addr, etc.
  - `PKT3_CP_DMA` is GFX6-only
  - `PKT3_DISPATCH_DIRECT` is for compute
  - `PKT3_DISPATCH_INDIRECT` is for compute
  - `PKT3_DISPATCH_MESH_INDIRECT_MULTI` is for mesh
  - `PKT3_DISPATCH_TASKMESH_DIRECT_ACE` is for mesh
  - `PKT3_DISPATCH_TASKMESH_GFX` is for mesh
  - `PKT3_DISPATCH_TASKMESH_INDIRECT_MULTI_ACE` is for mesh
  - `PKT3_DISPATCH_TASK_STATE_INIT` is for cmdbuf init
  - `PKT3_DMA_DATA` is dma between two addrs
  - `PKT3_DRAW_INDEX_2` is for draw
  - `PKT3_DRAW_INDEX_AUTO` is for draw
  - `PKT3_DRAW_INDEX_INDIRECT` is for draw is for draw
  - `PKT3_DRAW_INDEX_INDIRECT_MULTI` is for draw
  - `PKT3_DRAW_INDEX_OFFSET_2` is for draw
  - `PKT3_DRAW_INDIRECT` is for draw
  - `PKT3_DRAW_INDIRECT_MULTI` is for draw
  - `PKT3_EVENT_WRITE` is for misc events
  - `PKT3_EVENT_WRITE_EOP` is for GFX6-8
    - end-of-pipeline?
  - `PKT3_EVENT_WRITE_EOS` is for GFX6-8
  - `PKT3_INDEX_BASE` sets the index buffer
  - `PKT3_INDEX_BUFFER_SIZE` sets the index buffer size
  - `PKT3_INDEX_TYPE` sets the index type (before GFX9)
  - `PKT3_INDIRECT_BUFFER_CIK` is for `VK_NV_device_generated_commands`
  - `PKT3_LOAD_CONTEXT_REG_INDEX` loads ctx reg from addr (since newer GFX8)
    - same as `PKT3_COPY_DATA` followed by `PKT3_PFP_SYNC_ME`
  - `PKT3_LOAD_SH_REG_INDEX` loads sh reg from addr (since newer GFX8)
  - `PKT3_NOP` is nop
  - `PKT3_NUM_INSTANCES` sets the draw instance count
  - `PKT3_PFP_SYNC_ME` lets PFP waits for ME
    - e.g., `PKT3_COPY_DATA` or `PKT3_DMA_DATA` are executed by ME but index
      buffer is read by PFP
  - `PKT3_RELEASE_MEM` is for semaphore?
  - `PKT3_SET_BASE` is for indirect draw/dispatch/mesh
  - `PKT3_SET_CONFIG_REG` is for GFX6
  - `PKT3_SET_CONTEXT_REG` sets a ctx reg
  - `PKT3_SET_PREDICATION` is for conditional rendering
  - `PKT3_SET_SH_REG` sets a sh reg
  - `PKT3_SET_SH_REG_INDEX` sets a sh reg (since GFX10)
  - `PKT3_SET_UCONFIG_REG` sets a uconfig reg
  - `PKT3_STRMOUT_BUFFER_UPDATE` is for transform feedback
  - `PKT3_SURFACE_SYNC` is deprecated by `PKT3_ACQUIRE_MEM`
  - `PKT3_WAIT_REG_MEM` waits until reg or addr matches the ref val
  - `PKT3_WRITE_DATA` inline-writes vals to addr

## FMASK

- FMASK is optional and is used for MSAA compression up to GFX10
  - it is removed from GFX11+
  - as a color buffer, it requires `S_028C70_COMPRESSION`
  - as a texture, it requires a separate descriptor
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
  - for non-msaa surfaces, it can be enabled for fast clears up to GFX9
  - for msaa surfaces, it can be enabled for fmask compression up to GFX10
  - it is removed from GFX11+
  - as a color buffer, it requires `S_028C70_FAST_CLEAR`
    - if tc-compatible, it requires `S_028C70_FMASK_COMPRESS_1FRAG_ONLY` as
      well
  - as a texture, it requires `S_00A018_COMPRESSION_EN` or
    `S_008F28_COMPRESSION_EN`
    - it is not enabled for single-sampled textures
    - it is only enabled for the FMASK texture descriptors of multi-sampled
      textures that are TC-compatible
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

## HTILE

- HTILE is used for
  - hiz
  - stencil?
- TC compat
  - HTILE can be TC-compat on GFX8+
  - TC supports a subset of swizzles on GFX8/GFX9
  - TC supports all swizzles on GFX10+
- DB
  - `DB_Z_INFO`'s `TILE_SURFACE_ENABLE` controls whether htile is enabled for
    depth
  - `DB_STENCIL_INFO`'s `TILE_STENCIL_DISABLE` controls whether htile is
    disabled for stencil
    - this allocates the entire htile to hiz, allowing higher z-range
      precision
  - in general, we want to set `TILE_STENCIL_DISABLE` only when the image is
    depth-only
    - on gfx8, there is a hw bug and TC does not support higher z-range
      precision; we want additionally check that htile is not TC-compat
- HTILE format
  - each dword covers a block
    - block size depends on several factors
  - when `TILE_STENCIL_DISABLE`, the dword consists of
    - bit0..3: zmask
    - bit4..17: min z
    - bit18..31: max z
  - otherwise, if vrs is disabled,
    - bit0..3: zmask
    - bit4..5: sr0 for stencil result (b11 is unknown)
    - bit6..7: sr1 for stencil result
    - bit8..9: smem
    - bit10..11: unused
    - bit12..31: zrange
  - when a d+s image is initialized, htile is initialized to 0xfffff3ff
    - zmsk is 0xf
    - sr is 0xf (unknown)
    - smem is 0x3
    - z range is 0xfffff
- HTILE-related registers
  - `DB_DEPTH_CLEAR` is the depth fast clear value, which is used only when
    `ZMASK` of the htile is 0
  - `DB_STENCIL_CLEAR` is the stencil fast clear value, which is used when
    `SMEM` of the htile is 0
  - `DB_Z_INFO`
    - `ZRANGE_PRECISION`: when 0, `zrange` means `zmin`; when 1, `zrange`
      means `zmax`
- addrlib
  - `Addr2ComputeHtileInfo` computes the htile info

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
  - RB
    - render backend
- more terms
  - ACE
    - Asynchronous Compute Engines
  - RB+
    - reder backend plus (GFX10.3 or GFX9 APU or GFX8 stoneyridge)
  - SE
    - shader engine
  - PRT
    - partially resident texture
  - SA
    - shader array
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

## addrlib

- `addrinterface.h` is the public interface
  - `AddrCompute*` is for GFX8-
  - `Addr2Compute*` is for GFX9+
- `AddrCreate` calls `Lib::Create`
  - for GFX6, `SiHwlInit`
  - for GFX7-8, `CiHwlInit`
  - for GFX9, `Gfx9HwlInit`
  - for GFX10, `Gfx10HwlInit`
  - for GFX11, `Gfx11HwlInit`
  - `m_se` is the number of shader engines (pipelines)
  - `m_rbPerSe` is the number of render backends per shader engine
  - `m_pipes`
  - `m_banks`

## addrlib Tilings

- tilings
  - gfx8- uses `AddrTileMode` and gfx9+ uses `AddrSwizzleMode`
  - there is linear
    - `ADDR_TM_LINEAR_ALIGNED` on gfx8- or `ADDR_SW_LINEAR` on gfx9+
  - there are micro tiles
    - such as `ADDR_TM_1D_TILED_THIN1` on gfx8- or `ADDR_SW_256B_S` on gfx9+
    - each micro tile is 256 bytes
    - the micro tile size is
      - 8x8 on gfx8-
      - dynamic on gfx9+
    - there are also micro tile modes
      - S is standard, or `ADDR_NON_DISPLAYABLE` on gfx8-
      - D is displayable, or `ADDR_DISPLAYABLE` on gfx8-
      - R is rotated (gfx9) or renderable (gfx10+), or `ADDR_ROTATED` on gfx8-
      - Z is for depth/stencil/fmask, or `ADDR_DEPTH_SAMPLE_ORDER` on gfx8-
    - there are also variations
      - X is xor
      - T is prt (partially resident texture)
  - there are macro tiles
    - such as `ADDR_TM_2D_TILED_THIN1` on gfx8- or `ADDR_SW_4KB_S` on gfx9+
    - each macro tile is 4KB, 64KB, 256KB, or variable
    - within each macro tile, there are micro tiles that are 256 bytes
- `AddrSwizzleMode` enumerates all swizzle modes for GFX9+
  - they can be classifed by block types
    - `AddrBlockLinear` is linear
    - `AddrBlockMicro` uses 256B block
    - `AddrBlockThin4KB` uses thin 4KB block
    - `AddrBlockThick4KB` uses thick 4KB block
    - `AddrBlockThin64KB` thin 64KB block
    - `AddrBlockThick64KB` uses thick 64KB block
    - `AddrBlockThinVar` uses thin var block
    - `AddrBlockThickVar` uses thick var block
  - they can be classifed by swizzle modes
    - `sw_Z` is for z/s/fmask
    - `sw_S` is a standard swizzle defined by MS (microsoft?)
    - `sw_D` is compatible with display
    - `sw_R` is compatible with display rotation on GFX9 and is for render
      targers on GFX9+
  - `_X` and `_T` ones support "tile swizzle"
    - when the swizzle mode is `_X` or `_T`, it further supports tile swizzle,
      or "pipe/bank xor"
    - the "tile swizzle" is specified in the lower bits of the va address, and
      picks a different pipe xor and bank xor
    - basically when doing MRTs, we want all MRTs to be `_X`/`_T` and have
      differnent tile swizzles to maximize memory throughput
- `Addr2GetPreferredSurfaceSetting` calls
  `V2::Lib::Addr2GetPreferredSurfaceSetting`
  - this is called to guess the optimal tiling
  - for a plain `VK_FORMAT_B8G8R8A8_UNORM`,
    - `flags` has `color`, `texture`, and `opt4space`
    - `resourceType` is `ADDR_RSRC_TEX_2D`
    - `format` is `ADDR_FMT_32`
      - it is derived from cpp
    - `resourceLoction` is `ADDR_RSRC_LOC_INVIS`
      - invisible gpu heap
      - it seems unused and mesa always use it
    - `forbiddenBlock` has `micro`
      - this disallows micro (256b) blocks
      - the other blocks are linear or macro (4kb, 64kb, 256kb, etc.)
    - `numSamples` and `numFrags` are both 1
      - AMD has a tech called EQAA that allows the storage samples
        (`numFrags`) to be less than the rasterization samples (`numSamples`)
      - EQAA can be enabled with apps knowing about it
      - in radv, `numFrags` is always equal to `numSamples`
  - on GFX9, it calls `Gfx9Lib::HwlGetPreferredSurfaceSetting`
    - `ADDR2_SWMODE_SET` is mapped from `ADDR2_BLOCK_SET` 
      - `Gfx9LinearSwModeMask` from `linear`
      - `Gfx9Blk256BSwModeMask` from `micro`
      - `Gfx9Blk4KBSwModeMask` from `macroThin4KB`
      - `Gfx9Blk64KBSwModeMask` from `macroThin64KB`
      - it looks like thin is for any dim but thick is only for 3D
    - the allowed swizzle mask is masked by `Gfx9Rsrc2dSwModeMask`
      - `Gfx9Rsrc2dSwModeMask` should be the valid swizzles for 2d
    - if there are more than 1 block types allowed, pick the largest block
      type (e.g., 64KB on GFX9)
    - if there are more than 2 swizzle types allowed, prefer D over S over Z

## addrlib layout

- `Addr2ComputeSurfaceInfo` calls `V2::Lib::ComputeSurfaceInfo`
  - on GFX9, it calls `Gfx9Lib::HwlComputeSurfaceInfoLinear` or
    `Gfx9Lib::HwlComputeSurfaceInfoTiled`
    - it requires 256-byte alignment
  - for a plain `VK_FORMAT_B8G8R8A8_UNORM`,
    - note that `Addr2GetPreferredSurfaceSetting` should have been called to
      determine the swizzle mode
    - `ComputeBlockDimensionForSurf` returns the swizzle block dimension
      - for example, for 64KB block types and 32cpp formats, it picks 128x128
        for the block size (`128*128*4` is 64KB)
    - `pOut`
      - `blockWidth`, `blockHeight`, and `blockSlices` are the dimension of
        the swizzle block
      - `pitch` is in pixels and is aligned to at least `blockWidth`
      - `height` is in pixels and is aligned to `blockHeight`
      - `pMipInfo` is for the entire mipmap
      - `surfSize` is the size of the surface (entire mipmap)
      - `baseAlign` is the alignment of the memory base addr
        - if `ADDR_SW_*_X`, where X stands for XOR, the alignment is the
          swizzle block size
  - for a mipmapped `VK_FORMAT_B8G8R8A8_UNORM`,
    - `pIn->numMipLevels > 1`
    - `Gfx9Lib::HwlComputeSurfaceInfoTiled`
      - `Gfx9Lib::GetMipChainInfo` computes the mip chain info
        - `inTail` is set to true when the level is smaller than `tailMaxDim`
          - when that happens, the level is aligned to `tailMaxDim`
          - all levels smaller than 256 bytes are aligned to `Block256_2d`
        - the layout of each level is saved to `pMipInfo`
        - `offset` is incremented after each level
        - `mipPitch` and `mipHeight` are reduced in half after each level 
      - `Gfx9Lib::GetMipStartPos` computes the starting position of each
        level
        - mip1+ is on the right of mip0 when `ADDR_MAJOR_Y`
        - mip1+ is on the bottom of mip0 when `ADDR_MAJOR_X`
  - for an array `VK_FORMAT_B8G8R8A8_UNORM`,
    - `pIn->numSlices > 1`
    - `Gfx9Lib::HwlComputeSurfaceInfoTiled`
      - the levels of the mipchain are tightly packed and each slice consists
        of a complete mipchain
  - for a 3d `VK_FORMAT_B8G8R8A8_UNORM`,
    - `pIn->resourceType == ADDR_RSRC_TEX_3D` and `pIn->numSlices > 1`
    - `Gfx9Lib::HwlComputeSurfaceInfoTiled`
      - `pIn->numSlices` and `pOut->numSlices` is the depth of the 3d surface
      - `Gfx9Lib::GetMipChainInfo` packs slices of a level together
  - for a msaa `VK_FORMAT_B8G8R8A8_UNORM`,
    - `pIn->numSamples > 1`
      - `pIn->numFrags` is equal to `pIn->numSamples` and is the number of
        samples stored in memory
    - `pOut->sliceSize` is multipled by `pIn->numFrags`
      - that is, for each sample N, the levels of the mipchain of sample N are
        tightly packed to form a subslice
      - subslices are packed together to form a slice
- `AddrComputeSurfaceInfo` calls `V1::Lib::ComputeSurfaceInfo`
  - note that it computes a single level of the mip chain at a time
    - `ADDR_COMPUTE_SURFACE_INFO_INPUT::numMipLevels` is 0 and only
      `ADDR_COMPUTE_SURFACE_INFO_INPUT::mipLevel` is set to the current level
    - this is different from GFX9+, where
      `ADDR2_COMPUTE_SURFACE_INFO_INPUT::numMipLevels` is set to compute the
      entire mipchain
  - on GFX8, it calls `CiLib::HwlComputeSurfaceInfo`
    - which calls `SiLib::HwlComputeSurfaceInfo`
    - which calls `EgBasedLib::HwlComputeSurfaceInfo`
  - `EgBasedLib::ComputeSurfaceAlignmentsLinear` returns 3 alignments for
    base, pitch, and height respectively
  - for a plain `VK_FORMAT_B8G8R8A8_UNORM`,
  - for a mipmapped `VK_FORMAT_B8G8R8A8_UNORM`,
    - the caller should call this function on each level manually
  - for an array `VK_FORMAT_B8G8R8A8_UNORM`,
    - `pOut->surfSize` is multiplied by `expNumSlices`
  - for a 3d `VK_FORMAT_B8G8R8A8_UNORM`,
    - addrlib does not know if the surface is 2d or 3d, and a 3d surface is
      the same as an 2d array
  - for a msaa `VK_FORMAT_B8G8R8A8_UNORM`,
    - `bytesPerSlice` is multiplied by `numSamples`

## Image Descriptors

- an image descriptor is called a `SQ_IMG_RSRC`
- hw limits
  - mipmapping: 1d, 2d, 3d
  - array: 1d, 2d
  - msaa: 2d
    - no mipmapping and no filtering (`texture`)
    - only sample fetching (`texelFetch`)
- according to `gfx10-rsrc.json`, gfx10.3 has
  - `SQ_IMG_RSRC_WORD0`: lo addr
  - `SQ_IMG_RSRC_WORD1`:
    - bit0..7: hi addr (40-bit addr)
    - bit8..19: min lod
    - bit20..28: format
    - bit30..31: lo width
  - `SQ_IMG_RSRC_WORD2`:
    - bit0..11: hi width
    - bit14..27: height
    - bit30: unused
    - bit31: resource level
  - `SQ_IMG_RSRC_WORD3`:
    - bit0..2: dst sel x (channel swizzle)
    - bit3..5: dst sel y
    - bit6..8: dst sel z
    - bit9..11: dst sel w
    - bit12..15: base level (view first level)
      - if msaa, unused and mbz
    - bit16..19: last level (view last level)
      - if msaa, `log2(samples)` instead
    - bit20..24: sw mode (tiling)
    - bit25..27: bc swizzle (border color channel swizzle)
    - bit28..31: type (1d, 2d, cube, array, msaa, etc.)
  - `SQ_IMG_RSRC_WORD4`:
    - bit0..12: depth
      - if not 3d and not array, it's 0 or lo pitch
      - if not 3d, it's view last layer
      - if 3d, it's image depth instead (there is no 3d array)
    - bit13: hi pitch
    - bit16..28: base array (view first layer)
  - `SQ_IMG_RSRC_WORD5`:
    - bit0..3: array pitch
      - 0: base array and depth are wrt to mip 0 and cover the entire miptree
      - 1: base array and depth are wrt to base level and cover just the level
      - we want to use 0 in most cases
      - we want to use 1 only for 3D UAV or 2d-view-of-3d
    - bit4..7: max mip (image last level)
      - if msaa, `log2(samples)` instead
    - bit8..19: min lod warn
    - bit20..22: perf mod
    - bit23: corner samples
    - bit25: lod hdw cnt en
    - bit26: prt default
    - bit31: big page
  - `SQ_IMG_RSRC_WORD6`:
    - bit0..7: counter bank id
    - bit8..9: llc noalloc
    - bit10: iterate 256
    - bit15..16: max uncompressed block size (for dcc)
    - bit17..18: max compressed block size (for dcc)
    - bit19: meta pipe aligned (for fmask)
    - bit20: write compress en (for storage image dcc)
    - bit21: compression en (for dcc or cmask)
    - bit22: alpha is on msb
    - bit23: color transform
    - bit24..31: lo metadata addr (for dcc or cmask)
  - `SQ_IMG_RSRC_WORD7`: hi metadata addr

## RDNA3 Instruction Set

- <https://gpuopen.com/amd-isa-documentation/>
- <https://www.amd.com/system/files/TechDocs/rdna3-shader-instruction-set-architecture-feb-2023_0.pdf>
- 3.3. Storage State: SGPR, VGPR, LDS
  - each wave is allocated 106 normal SGPRs
  - VGPRs are allocated in blocks of 16 for wave32 or 8 for wave64, and a
    shader may have up to 256 VGPRs.
- 16.1. SOP2 Instructions
  - `S_ADD_U32`
- 16.2. SOPK Instructions
  - `S_MOVK_I32`
- 16.3. SOP1 Instructions
  - `S_MOV_B32`
- 16.4. SOPC Instructions
  - `S_CMP_EQ_I32`
- 16.5. SOPP Instructions
  - `S_NOP`
- 16.6. SMEM Instructions
  - Scalar Memory Loads (SMEM) instructions allow a shader program to load
    data from memory into SGPRs through the Constant Cache ("Kcache").
    Instructions can load from 1 to 16 DWORDs. Data is loaded directly into
    SGPRs without any format conversion.
  - `S_LOAD_B32` reads 1 dword
    - addr is in base+offset
- 16.7. VOP2 Instructions
  - `V_CNDMASK_B32`
  - `V_ADD_NC_U32` Add two unsigned integers. No carry-in or carry-out.
- 16.8. VOP1 Instructions
  - `V_NOP`
- 16.9. VOPC Instructions
  - `V_CMP_F_F16`
- 16.10. VOP3P Instructions
  - `V_PK_MAD_I16`
- 16.11. VOPD Instructions
  - The VOPD encoded describes two VALU opcodes that are executed in
    parallel.
  - `V_DUAL_FMAC_F32`
- 16.12. VOP3 & VOP3SD Instructions
  - `V_NOP`
- 16.13. VINTERP Instructions
  - Parameter interpolation VALU instructions.
  - `V_INTERP_P10_F32`
- 16.14. Parameter and Direct Load from LDS Instructions
  - These instructions load data from LDS into a VGPR where the LDS address is
    derived from wave state and the M0 register.
  - `LDS_PARAM_LOAD`
- 16.15. LDS & GDS Instructions
  - This suite of instructions operates on data stored within the data share
    memory. The instructions transfer data between VGPRs and data share
    memory.
  - `DS_ADD_U32`
- 16.16. MUBUF Instructions
  - `BUFFER_LOAD_FORMAT_X`
- 16.17. MTBUF Instructions
  - `TBUFFER_LOAD_FORMAT_X`
- 16.18. MIMG Instructions
  - `IMAGE_LOAD`
- 16.19. EXPORT Instructions
  - Transfer vertex position, vertex parameter, pixel color, or pixel depth
    information to the output buffer
  - `EXP`
- 16.20.1. Flat Instructions
  - Flat instructions look at the per work-item address and determine for each
    work-item if the target memory address is in global, private or scratch
    memory.
  - `FLAT_LOAD_U8`
- 16.20.2. Scratch Instructions
  - Scratch instructions are like Flat, but assume all work-item addresses
    fall in scratch (private) space.
  - `SCRATCH_LOAD_U8`
- 16.20.3. Global Instructions
  - Global instructions are like Flat, but assume all work-item addresses fall
    in global memory space.
  - `GLOBAL_LOAD_U8`

## Vega Instruction Set

- <https://www.amd.com/system/files/TechDocs/vega-7nm-shader-instruction-set-architecture.pdf>
- 8.1.8. Buffer Resource
  - The buffer resource describes the location of a buffer in memory and the
    format of the data in the buffer. It is specified in four consecutive
    SGPRs (four aligned SGPRs) and sent to the texture cache with each buffer
    instruction.
- 12.1. SOP2 Instructions
- 12.2. SOPK Instructions
  - `S_MOVK_I32`
- 12.3. SOP1 Instructions
  - `S_MOV_B32`
- 12.4. SOPC Instructions
- 12.5. SOPP Instructions
  - `S_ENDPGM` End of program; terminate wavefront.
  - `S_WAITCNT` Wait for the counts of outstanding lds, vector-memory and
    export/vmem-write-data to be at or below the specified levels.
  - `S_SETPRIO`  User settable wave priority
- 12.6. SMEM Instructions
  - Scalar Memory Read (SMEM) instructions allow a shader program to load data
    from memory into SGPRs through the Scalar Data Cache, or write data from
    SGPRs to memory through the Scalar Data Cache.
    - `SBASE` SGPR-pair which provides a base address
    - `OFFSET` offset
  - `S_LOAD_DWORD`  Read 1 dword from scalar data cache
  - `S_LOAD_DWORDX4` 
  - `S_BUFFER_LOAD_B32` Read 1 dword from scalar data cache.
- 12.7. VOP2 Instructions
  - `V_ADD_U32`
  - `V_AND_B32`
  - `V_MUL_F32` Multiply two single-precision values.
- 12.7.1. VOP2 using VOP3 encoding
  - `V_ADD_U32` with alternative encoding for extra controls
- 12.8. VOP1 Instructions
  - `V_CVT_F32_I32` Convert from a signed integer to a single-precision float.
- 12.9. VOPC Instructions
- 12.10. VOP3P Instructions
- 12.11. VINTERP Instructions
- 12.12. VOP3A & VOP3B Instructions
  - `V_MED3_F32` Return median single precision value of three inputs.
- 12.13. LDS & GDS Instructions
- 12.14. MUBUF Instructions
  - Buffer instructions (MTBUF and MUBUF) allow the shader program to read
    from, and write to, linear buffers in memory.
  - `BUFFER_LOAD_DWORD` Untyped buffer load dword
    - `IDXEN` Supply an index from VGPR
    - `OFFEN` Supply an offset from VGPR
- 12.15. MTBUF Instructions
- 12.16. MIMG Instructions
- 12.17. EXPORT Instructions
  - `EXP` Transfer vertex position, vertex parameter, pixel color, or pixel
    depth information to the output buffer. 
- 12.18.1. Flat Instructions
- 12.18.2. Scratch Instructions
- 12.18.3. Global Instructions
