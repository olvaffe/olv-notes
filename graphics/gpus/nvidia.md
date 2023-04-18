nVidia
======

## History

- NV1 released in 1995
- NV3 released in 1997
  - RIVA 128
  - Direct3D 5.0
- NV4 released in 1998
  - RIVA TNT
  - Direct3D 6.0
  - second pixel pipeline doubling the rendering speed
  - 32-bit pixel formats
  - 24-bit depth buffer and 8-bit stencil
  - dual texture and bilinear filtering
  - 1024x1024 textures
- NV5 released in 1999
  - RIVA TNT2
  - Direct3D 6.0
  - 2048x2048 textures
- NV10 released in 1999
  - GeForce 256
  - Direct3D 7.0
  - hardware TNL
  - cube mapping
  - dot product bump mapping
  - 2x anisotropic and trilinear filtering
  - texture compression
  - four 64-bit pixel pipelines
- NV15 released in 2000
  - GeForce 2
  - Direct3D 7.0
  - two texture units per pixel pipeline
- NV20 released in 2001
  - GeForce 3
  - Direct3D 8.0 w/ shader model 1.1
  - 3D textures
  - shadow maps
  - 8x anisotropic filtering
  - MSAA
- NV25 released in 2002
  - GeForce 4
  - Direct3D 8.0a w/ shader model 1.1/1.3
  - second vertex pipeline
- NV30 released in 2003
  - GeForce FX
  - Direct3D 9.0a w/ shader model 2.0a
    - longer shaders and more ops
  - better filtering and multisampling
- NV40 released in 2004
  - GeForce 6
  - Direct3D 9.0c w/ shader model 3.0
    - more flow controls
  - vertex texture fetch
  - 64-bit HDR
  - multi gpu
  - 16 pixel pipelines
- G7x released in 2005
  - GeForce 7
  - Direct3D 9.0c w/ shader model 3.0
- G8x released in 2006
  - GeForce 8
  - Direct3D 10.0 w/ shader model 4.0
    - geometry shader
    - unified shader
  - Tesla architecture
  - unified shader architecture
  - 64 stream processors; scalar only, no vector
  - CUDA
  - high quality texture filtering
  - MSAA/CSAA
  - 128-bit HDR
- G9x released in 2008
  - GeForce 9
  - Direct3D 10.0 w/ shader model 4.0
  - Tesla architecture
- GT200 released in 2008
  - GeForce 200
  - Direct3D 10.1 w/ shader model 4.0
  - Tesla architecture
- GF10x released in 2010
  - GeForce 400
  - Direct3D 11.0 (or 12.0 with `11_0` feature level)
  - Fermi architecture
  - 16 SMs (stream multiprocessors) of 32 cuda cores
- GF11x released in 2010
  - GeForce 500
  - Direct3D 11.0
  - Fermi architecture
- GK10x released in 2012
  - GeForce 600
  - Direct3D 11.0
  - Kepler architecture
  - energy efficiency
  - a SM has 192 cores
  - bindless textures
- GK11x released in 2013
  - GeForce 700
  - Direct3D 11.0
  - Kepler architecture
  - compute performance
- GM10x released in 2014
  - GeForce 900
  - Direct3D 12.0 with `12_1` feature level
  - Maxwell architecture
  - a SM has 128 cores
- GP10x released in 2016
  - GeForce 10
  - Direct3D 12.0 with `12_1` feature level
  - Pascal architecture
  - a SM has 128 cores
  - HBM, high bandwidth memory
  - unified memory w/ page migration engine
  - NVLink
  - raytracing
- TU10x released in 2018
  - GeForce 20
  - Direct3D 12.0 with `12_2` feature level
  - Turing architecture
  - cuda cores
  - raytracing cores
  - tensor cores
- GA10x released in 2020
  - GeForce 30
  - Direct3D 12.0 Ultimate with `12_2` feature level
  - Ampere architecture
- AD10x released in 2020
  - GeForce 40
  - Direct3D 12.0 Ultimate with `12_2` feature level
  - Ada Lovelace architecture

## Naming

- e.g., `GeForce RTX 4070 Ti`
- `GeForce RTX`
  - before `20` series, it was `GTX`
  - since `20` series, it is `RTX` and indicates raytracing
- generations
  - it was `1xx`, `2xx`, until `9xx`
  - it is `10xx`, `20xx`, and is `40xx` currently
- performances
  - within a generation, the last 2 numbers indicate the performance levels
  - `50` is entry-level, $2xx
  - `60` is mid-range, $3xx
  - `70` is high-end, $6xx
  - `80` and `90` are enthusiast, $1xxx
- suffices
  - `Ti` is the faster variant
    - it stands for `Titanium`
    - `4070 < 4070 Ti < 4080`
  - `Super` is similar but is slower than `Ti`

## Turing

- nVidia Turing TU102 GPU
- $999
- 6 Graphics Processor Clusters, each has
  - 1 Raster Engine
    - Edge Setup
    - Rasterization
    - Z-Cull
  - 2 32-bit GDDR6 memory controllers, each has
    - 8 ROP units
    - 1 512KB L2
  - 6 Texture Processing Clusters, each has
    - 1 PolyMorph Engine
      - Vertex Fetch
      - Tessellation
      - Viewport Transform
      - Attribute Setup
      - Stream Output
    - 2 Streaming Multiprocessors (SMs)
- a SM has
  - 1 96KB configurable shared memory / L1
  - 4 texture units
  - 1 RT Core
  - 4 Processing Blocks, each has
    - 1 Dispatch Unit
    - 1 Warp Scheduler
    - 1 64KB Register File
    - 1 L0
    - 16 FP32 CUDA Cores + accompanying INT32 Cores
    - 2 Tensor Cores
    - 4 LD/ST Units
      - memory access
    - 1 SFU
      - sin/cos/etc

## Life of a Triangle

- insert a draw command into the pushbuffer
- ask GPU Host Interface to pick up the pushbuffer which is processed by Front
  End
- Primitive Distributor processes the indices in the index buffer and
  distribute triangle work batches to GPCs (Graphics Processor Clusters)
- Poly Morph Engine takes care of fetching vertex data
- after the vertex data has been fetched, warps of 32 threads are scheduled
  inside the SM
- the warp scheduler decides which warp to run and which instruction to issue
- warps advance in lock-step; when a warp is waiting for slow operations such
  as memory load, the scheduler may pick another warp to run.
  Context-switching is cheap because each warp has its own registers in the
  register file.  The more registers a shader need, the less concurrent warps
  there can be.
- Viewport Transform
- bin triangles into tiles and send tiles to GPCs via Work Distribution
  Crossbar
- Attribute Setup turns vertex data into a more friendly format
- Raster Engine does back-face culling and z-culling; then generate pixel
  information
- once there are 8 2x2 pixel quads, the warp scheduler will manage the task
- the color and depth values are sent to ROP, which does depth testing,
  blending, etc.
