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
