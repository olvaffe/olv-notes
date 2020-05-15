GPU
===

## CPU vs. GPU

- a CPU core has
  - CUs
  - ALUs
  - L1
- a GPU core devotes most of itself only to ALUs
- start with a scalar CPU core
  - there is one CU and one EU
  - observe that we don't necessarily need cache, out-of-order, branch
    predictor, memory pre-fetcher
  - the core can be simpler, more efficient, and more affordable
- let's have 16 cores in our GPU
  - observe that the cores often execute the same instruction stream and the
    bottleneck is the EUs
  - a core should have multiple EUs and multiple register files sharing the
    same CU
- let's have 8 EUs in each core
  - this is SIMT (single instruction, multiple threads) because there is only
    one CU and all 8 threads on 8 EUs must execute in lock-step
  - wasted EUs if some of them are inactive because of branching
  - stalled CU/EUs if some of them execute high latency operation (e.g.,
    texturing)
  - an EU should have multiple register files to context switch between
    multiple instruction streams
- let's have 8 register files for each EU
  - context switch (super-threading) between 8 8-instruction-streams
  - now we can execute `16*8*8*8` = 8192 threads concurrently.
  - some threads are interleaved
  - on some implementations, max threads depends on the instruction stream.
    - when an instruction stream uses only half of the registers, there can be
      8192*2 threads concurrently.
- Fermi (2010)
  - a core has 2 CUs and 32 EUs; each EU can interleave 48 instruction streams
  - there are 15 cores: `15*32*48` = 23040 threads!
- Turing (2018)
  - each core has 4 CUs and 64 EUs; each EU can interleave 128 instruction streams
  - there are 68 cores: `68*64*128` = 557056 threads

## Terminology

- nVidia / AMD / Intel / ARM / Qualcomm
- Turing / RDNA / Gen11 / Valhall / A6XX
- SMs / CUs / SubSlices / Shader Cores / ?
  - count: 72 / 20 / 8 / 1 / 1 / ?
- Processing Blocks / SIMDs / EUs / Execution Engines / Shader Processor
  - count per SM/CU/SubSlice/ShaderCore: 4 / 4 / 8 / 1 / ?
- CUDA cores / ALUs / 4-wide SIMD ALUs / 16-wide Processing Units / ?
  - count per unit: 16 / 32 / 2 / 2 / ?

## Memory Bandwidth

- DRAM is asynchronous
- SDRAM is synchronous and relies on an external clock
  - for each cycle, the voltage goes high and keeps there for a bit before it
    goes low.  I guess it can be filterd such that the leading and trailing
    edges are cycle/2 apart
- DDR SDRAM transfers data on leading and trailing edges
- Internal Clock Rate / prefetch buffer size / External Clock Rate
  - SDR :  66-133 MHz /  1n / internal*1
  - DDR : 100-200 MHz /  2n / internal*2/2 (divide by two because of two edges)
  - DDR2: 100-200 MHz /  4n / internal*4/2
  - DDR3: 100-266 MHz /  8n / internal*8/2
  - DDR4: 200-400 MHz /  8n / internal*8/2
  - DDR5: 200-400 MHz / 16n / internal*16/2
- LPDDR
  - same as DDR
- GDDR
  - like DDR @ higher frequency
  - GDDR2 : 4n-prefetch @ 500 Mhz
  - GDDR3 : 4n-prefetch @ 625 MHz
  - GDDR4 : 8n-prefetch @ 275 MHz
  - GDDR5 : 8n-prefetch @ 625-1000 Mhz
  - GDDR5X: 16n-prefetch @ 625-875 Mhz
  - GDDR6 : 16n-prefetch @ 875-1000 Mhz
- DIMMs have a 64-bit data path
  - stack eight 8-bit chips together to provide 64-bit
  - SDR : 133 *  1 * 64 / 8 / 1024 =  1.0 GB/s
  - DDR : 200 *  2 * 64 / 8 / 1024 =  3.2 GB/s
  - DDR2: 200 *  4 * 64 / 8 / 1024 =  6.4 GB/s
  - DDR3: 200 *  8 * 64 / 8 / 1024 = 12.8 GB/s
  - DDR4: 400 *  8 * 64 / 8 / 1024 = 25.6 GB/s
  - DDR5: 400 * 16 * 64 / 8 / 1024 = 51.2 GB/s
- GPU uses various data path widths
  - Radeon RX 5700 XT is 875MHz GDDR6, eight 32-bit channels,
    - `875 * 16 * (8 * 32) / 8 / 1024` = 448 GB/s
- HBM
  - 3D-stacked SDRAM dies with very high data path width
    - A DRAM die has two 128-bit channels 
    - A 4-Hi die (4 1-Hi dies stacked) has 8 128-bit channels
  - HBM: 2n-prefetch @ 500 MHz
    - with four 4-hi dies, the total width is `4 * 4 * 2 * 128 = 4096`
    - `500 * 2 * 4096 / 8 / 1024` = 500 GB/s
  - HBM2: 4n-prefetch @ 500 MHz
    - with four 8-hi dies, the total width is `4 * 8 * 2 * 128 = 8192`
    - `500 * 4 * 8192 / 8 / 1024` = 2000 GB/s
 
## Mobile GPU Power Budget

- depends on heat dissipation
  - a small smartphone: 1-2 Watts
  - a large smartphone: 2-3 Watts
  - a tablet: 4-5 Watts
  - an embedded device with heat sink: up to 10 Watts

## Tiler

- why?
  - SoC power budget is 2.5 to 3.5 Watts for most devices
  - This leaves 1 to 1.5 Watts for GPU
  - 1GB/s external memory access is about 100mW
  - to reduce external memory access,
    - tile-based rendering and on-chip local ram
    - compression
    - on Mali, a tile is 16x16
- a render pass is a sequence of draw calls using the same framebuffer
  - fb consists of multiple color/depth/stencil/resolve/etc attachments
  - a draw might use only a subset of the fb attachments
  - rendering 3D scene and 2D UI to a fb are logically two passses; they
    should be merged into one physical render pass
- immediate mode rendering (IR)
  - for draw in pass:
      for primitive in draw:
        for vertex in primitive:
          vs(vertex)
        for fragment in rasterize(primitive):
          fs(fragment)
  - vs outputs to a on-chip fifo
  - fs accesses the external memory for everything, including blending, depth
    test, and stencil test.
  - caches help
- tile-based rendering (TBR)
  - goal is to minimize external memory access
  - for draw in pass:
      for primitive in draw:
        for vertex in primitive:
          vs(vertex)
        tiler(primitive)
    for tile in framebuffer:
      for primitive in primitive_list(tile):
        for fragment in raterize(primitive):
          fs(fragment)
      flush();
  - vs outputs to a on-chip fifo
  - tiler outputs data to external memory, which describe which primitives
    touch which tiles
    - space needed is normally 1 bit per primitive per tile
    - with N primitives, each tile has N bits indicating whether each
      primitive touches the tile or not
  - because the tile is small, the fs working set (color, depth, and stencil
    buffers) can fit in a on-chip memory.  fs does not access the external
    memory for blending, depth test, and stencil test.  It still access the
    external memory for texturing.
  - when the tile is flushed to the external memory, it is also desirable to
    output compressed data
  - MSAA is also cheap because resolve can happen in flush
  - great save on deferred shading G-buffer
- there are also TBDR and TBIR
  - TBR above is TBDR.  TBIR is
  - for draw in pass:
      for primitive in draw:
        for vertex in primitive:
          vs(vertex)
        tiler(primitive)
      for tile in framebuffer:
        for primitive in primitive_list(tile):
          for fragment in raterize(primitive):
            fs(fragment)
        flush();
  - TBDR allows better hidden surface removal?
    - IR or TBIR only does as good as early-Z
- tile-based rendering disadvantages
  - tiler outputs to external memory and can be more expensive with simpler fs
  - with tessellation, tiler outputs are explosive
- on-chip memory load/store ops
  - when a tile begins, a load op is required to initialize the on-chip memory
    from a framebuffer attachment
    - one of `LOAD`, `CLEAR`, or `DONT_CARE`
  - when a tile ends, a store op is required to write back from the on-chip
    memory to a framebuffer attachment
    - `STORE` or `DONT_CARE`
  - when an attachment is transient, load op is usually `CLEAR` and store op
    is `DONT_CARE`; On Vulkan, use `VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT`
    and `VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT`  as well.

## Assets Best Practice

- Geometry
  - 32-bit index is optional in Vulkan; mobile supports 16-bit index meaning
    the maximum vertex count per mesh is 65536
  - vertices are expensive (in terms of energy due to memory access?) on
    mobile
  - small details should use normal map, not vertices
  - avoid long and thin triangles
    - they might touch/miss the sampling point with small camera movement,
      resulting in flickering
    - they are slower to render due to bad locality
  - use LODs for meashes as welll
- Texture
  - always mipmap
  - texture compression
  - altas
  - pack roughness/metallic/Ambient Occlusion/alpha into different channels
    of one texture

## CUDA

- `MatAdd<<<numBlocks, threadsPerBlock>>>(A, B, C)` adds matrices `A` and `B`,
  and stores the result in `C`
  - Usually, there are 16x16 threads per block
  - Say the size of a matrix is NxN, there will be `(N/16 * N/16)` blocks
  - each block must be run by a single core
  - different blocks may be run by different cores
- thread local memory is faster than shared memory is faster than the global
  memory
- CUDA hardware is built around an array of multithreaded SMs (streaming
  multiprocessors)
- When a SM is given one or more thread blocks to execute, it partitions them
  into warps that get scheduled by a warp scheduler
  - a warp of an SM has 32 threads

## Binding Models

- Fixed-Function
  - `glUniform1f(loc, val)` becomes `writeReg(REG_UNIFORM_<loc>,  val)`
  - `layout(location = loc) uniform float v` in GLSL becomes
    `float v = readReg(REG_UNIFORM_<loc>)`
  - `glActiveTexture(GL_TEXTURE0 + loc); glBindTexture(GL_TEXTURE_2D, tex)`
    becomes `writeRegs(REG_TEXTURE_<loc>, texParamsAndAddr)`
  - `layout(location = loc) sampler2D tex; vec4 color = texture(tex, texcoord);`
    becomes `vec4 color = sample(REG_TEXTURE_<loc>, texcoord)`
- Bindless
  - `glUniform1f(loc, val)` becomes `rootTable[UNIFORM_START + loc] = val`
  - `layout(location = loc) uniform float v` in GLSL becomes
    `float v = rootTable[UNIFORM_OFFSET + loc]`
  - `glActiveTexture(GL_TEXTURE0 + loc); glBindTexture(GL_TEXTURE_2D, tex)`
    becomes `memcpy(&rootTable[TEXTURE_OFFSET + TEXTURE_SIZE * loc,
    texParamsAndAddr, TEXTURE_SIZE)`
  - `layout(location = loc) sampler2D tex; vec4 color = texture(tex, texcoord);`
    becomes `vec4 color = sample(&rootTable[TEXTURE_OFFSET + TEXTURE_SIZE *
    loc, texcoord)`
  - It is also possible to have infinite levels of indirection
    `rootTable[N] = secondaryTable`
- Vulkan
  - `layout(set = S, binding = B) sampler2D textures[]; texture(textures[dynamic_index], texcoord)`
    becomes `sample(&rootTable[S][BINDING_B_OFFSET + TEXTURE_SIZE * dynamic_index], texcoord)`
  - a set is an array of bindings
    - each binding has an array of a descriptor type
    - layout(binding = B) uniform Type[count];

## MC (Memory Controller)

- GPU has its own address space
- It has access to its VRAM and system memory through GART
- Both need to be mapped into GPU's address space
- hardpin vs softpin
  - when a memory is allocated, it can be a shallow struct
  - just before GPU work that accesses the memory is scheduled,
    - pages are allocated
    - an address range is reserved
    - mapping is setup
  - this is called hardpin.  Only after the memory is hard pinned, the address
    of the memory is known.  The command stream needs to be patched to use the
    address.
  - if the GPU has a huge address space, the driver can instead reserve the
    address range when the memory is allocated.  This is called softpin.
    - page allocation and mapping setup can still be deferred
    - no more command stream patching is needed

## GART

- GART was introduced with AGP
  - the graphics device is called the master
  - the core-logic (northbridge) is called the target
  - an aperture is a contiguous range of the physical address space where
    master accesses are re-mapped by the target to physically non-contiguous
    system memory pages
  - the remapping is accomplished through GART, Graphics Aperture Re-mapping
    Table, living in the system memory
  - core-logic coherency
    - master accesses outside of the aperture must be coherent
      - master writes is visible to CPU
      - CPU writes is visible to master reads
    - master accesses inside the aperture depends on `ita_coh` and
      `gart_entry_coh`
      - when `gart_entry_coh` is set, `ita_coh` (read-only, core-logic cap)
      	decides the coherency
      - when `gart_entry_coh` is unset, coherency is undefined
- In PCI-e, GART is implemented by the graphics device using the VRAM, not by
  the core-logic

## Memory Types

- not cached, coherent
  - good for streaming CPU write-only resources
  - always supported
  - caches are not polluted
- cached, incoherent
  - use only when CPU read is needed
- cached, coherent
  - not always supported
  - small power cost over "not cached, coherent"

## Warps

- vector design (SIMD)
  - low utilization
- scalar design (SIMT)
  - high utilization
- warps
  - better cache utilization
  - save real estate on control logic for ALUs
  - 4-wide: arm bifrost 1st and 2nd gens
  - 8-wide: arm bifrost 3rd gen
  - 16-wide: arm valhall
  - 32-wide: nvidia, amd rdna
  - 64-wide: amd gcn, amd rdna
