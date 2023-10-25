ARM Mali
========

## History

- 2007: Utgard
  - GLES 2.0
  - non-unified, vector isa
  - Mali-{200,300,400,450}
- 2010: Midgard
  - GLES 3.1
  - unified, vector isa
  - 1 to 4 unified shader cores
  - 2 or 4 ALUs per shader core
  - 4x or 16x FSAA
  - 1 MMU
  - 32KB to 256KB L2
  - 16x16 tiles
  - Mali-{T604,T658}
- 2012: Midgard 2nd gen
  - ASTC
  - Mali-{T622,T624,T628,T678}
- 2013: Midgard 3nd gen
  - GLES 3.2
  - 4x to 16x MSAA
  - 256KB to 2048MB L2
  - 1 to 16 cores
  - AFBC framebuffer compression
  - Mali-{T720,T760}
- 2014: Midgard 4nd gen
  - Mali-{T820,T830,T860,T880}
- 2016: Bifrost
  - Unified shaders with quad vectorization
  - Scalar ISA
  - Clauses execution
  - 1 to 32 cores
  - Full cache coherency
  - Mali-{G31,G51,G71}
- 2017: Bifrost 2nd gen
  - Mali-{G52,G72}
- 2018: Bifrost 3nd gen
  - 8 execution lanes per engine (up from 4). Doubled pixel and texel
    throughput.
  - Mali-G76
- 2019: Valhall
  - New superscalar engine
  - Simplified scalar ISA
  - New dynamic scheduling
  - Mali-{G57,G77}
- 2020: Valhall 2nd gen
  - Asynchronous clock domains
  - New FMA units and increase Tiler throughput
  - Arm Frame Buffer Compression (AFBC) 1.3
  - Mali-{G68,G78}
- 2021: Valhall 3nd gen
  - Larger shader cores (2x compared to Valhall 2nd Gen)
  - New GPU frontend, Command Stream Frontend (CSF) replaces the Job Manager
  - Mali-{G310,G510,G610,G710}
- 2022: Valhall 4nd gen
  - Ray Tracing support (hardware-based)
  - Variable Rate Shading
  - New Execution Engine, with doubled the FMA block, Matrix Multiply instruction support, and PPA improvements
  - Arm Fixed Rate Compression (AFRC)
  - Mali-{G615,G715}, Immortalis-G715
- 2023: 5nd gen
  - Deferred vertex shading (DVS) pipeline
  - Mali-{G620,G720}, Immortalis-G720

## Identification

- `GPU_ID` register
  - higher 16 bits: id
  - lower 16 bits: revision
    - bit 0..3: status
    - bit 4..7: minor
    - bit 8..11: major
- `pan_arch()`
  - midgard: v4 and v5
    - the 16-bit id is less than `0x1000`
    - a table is used to look up the arch versions
  - bifrost: v6 and v7
    - `id >> 12` is the arch versions
  - valhall: v9

## Utgard

- GLES 2.0
- non-unified shader cores
- one vertex core
  - the vertex core and the tiler share a small L2
  - shade one or two vertices serially
  - output to fixed-function tiler for primitive assembly, clipping, cullling,
    and tile list generation
- multiple fragment cores
  - they share a larger L2
  - L2 size is usually `num_cores * 32KB`
  - data flow within a core
    - tile list reader
    - rasterizer
    - early zs tester
    - fragment thread creater
    - threads doing load, texture, arithmetic, and then retire
    - late zs tester
    - blender
    - tile memory
    - tile writeback
  - a core can run up to 128 threads to hide cache misses and memory latency
  - ISA is SIMD and operate on vec4 fp16

## Midgard

- GLES 3.2
- unified shader cores
- a L2 cache shared by all shader cores
  - size is `num_cores * (32KB or 64KB)`
- given a render pass
  - the driver submits geometry workload to HW first
  - the geometry queue block dispatches the workload to shader cores
  - the driver then submits fragment workload to HW
  - the fragment queue block dispatches the workload to shader cores
- a shader core consists of
  - fixed-function part
    - tile list reader
    - rasterizer
    - early zs tester
    - vertex or fragment thread creator
  - programmable tripipe
    - thread pool
    - three types of pipelines
      - one load/store pipeline w/ L1
      - one texture pipeline w/ L1
      - one or more arithmetic pipelines
	- 128-bit SIMD (4 fp32 or 8 fp16 or 16 i8)
    - thread retire
  - more fixed-function parts
    - late zs tester
    - blender
    - tile memory
    - tile writeback
- 16x16 tiles
- 4KB on-chip memory

## Bifrost

- "The Bifrost GPU architecture and the ARM Mali-G71 GPU"
  - by Jem Davies
- a L2 cache shared by all shader cores
  - size is `num_cores * (64KB or 128KB)`
  - full coherency
- up to 32 unified shader cores, where each core consists of
  - a compute frontend
    - quad creator
  - a fragment frontend
    - tile list reader
    - rasterizer
    - early zs tester
    - quad creator
  - a quad manager
    - warp manager
  - up to three execution engines, each
    - 32-bit scalar ISA
    - bifrost uses 4-lane quad-parallel vectorization (SIMT...)
      - a quad is 4 scalar threads executed in lockdep
      - one quad at a time executes in each pipeline stage
      - each thread fills a 32-bit lane of the hardware
      - 4 threads all doing a vec fp32 add take 3 cycles
    - midgard uses 4-lane SIMD vectorization
      - one thread at a time executes in each pipeline stage
      - each thread must fill the width of the hardware
      - 4 threads all doing a SIMD vec3 fp32 add take 4 cycles
  - load/store unit
  - attribute unit
  - varying unit
  - texture units (one or two)
  - zs/blend units (one or two)
    - late zs tester
    - blender
    - tile memory
    - tile writeback
- geometry pipeline
  - instead of 'shade -> assembly -> clip/cull`
  - it does 'assembly -> position shade -> clip/cull -> varying shade`
  - specifically,
    - tiler fetches indices for primitive assembly
    - position shading fetches and transforms only positions
    - tiler clips/culls and stores
      - transformed positions
      - polyton list 
    - varying shading fetches attributes and stores varyings
    - fraghment shading reads back
      - transformed positions
      - polyton list 
      - varyings
  - it is beneficial to have two packed buffers, one for positions and one for
    attributes

## Valhall

- G77 high-level features
  - better AR and ML
  - a new superscalar engine
  - a simplified scalar ISA that is more compiler-friendly
  - dynamic scheduling of instructions
  - data structures that are more Vulkan-friendly
  - 16-wide warps, 32 lanes
    - G76 has 8-wide warps, 24 lanes
  - quad texture mapper, 4 texels/cycle
    - G76 is 2 texels/cycle
  - AFBC 1.3
- a shader core consists of
  - a compute frontend
  - a fragment frontend
  - a manager
  - an execution engine
    - front-end
      - create/retire warps
      - track states for warps
    - scheduler
      - issue instructions
    - two processsing units, each
      - 16-wide FMA unit
      - 16-wide CVT unit (convert unit)
      - 4-wide special function unit
  - an attribute unit
  - a varying unit
  - 4x texture unit
  - a load-store cache
    - replaces the load-store unit
