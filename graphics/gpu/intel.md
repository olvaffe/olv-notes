Intel GPUs
==========

## uArchs

- Nehalem
  - Nehalem, 2008, 45nm, 1st gen, Gfx4
    - `intel_device_info_i965`, etc.
  - Westmere, 2010, 32nm, 1st gen, Gfx5
    - `intel_device_info_ilk`
- Sandy Bridge
  - Sandy Bridge, 2011, 32nm, 2nd gen, Gfx6
    - `intel_device_info_snb_gt1`, etc.
  - Ivy Bridge, 2012, 22nm, 3th gen, Gfx7
    - `intel_device_info_ivb_gt1`, `intel_device_info_byt`, etc.
- Haswell
  - Haswell, 2013, 22nm, 4th gen, Gfx7.5
    - `intel_device_info_hsw_gt1`, etc.
  - Broadwell, 2014, 14nm, 5th gen, Gfx8
    - `intel_device_info_bdw_gt1`, `intel_device_info_chv`, etc.
- Skylake
  - Skylake, 2015, 14nm, 6th gen, Gfx9
    - `intel_device_info_skl_gt1`, `intel_device_info_bxt`, etc.
  - Kaby Lake, 2016, 14nm, 7th gen, Gfx9.5
    - `intel_device_info_kbl_gt1`, `intel_device_info_glk`, etc.
  - Coffee Lake, 2017, 14nm, 8th/9th gen, Gfx9.5
    - `intel_device_info_cfl_gt1`
  - (SKIPPED) Cannon Lake, 2018, 10nm, 9th gen, Gfx10
  - Comet Lake, 2019, 14nm, 10th gen, Gfx9.5
- Ice Lake
  - Ice Lake, 2019, 10nm, 10th gen, Gfx11
    - `intel_device_info_icl_gt2`, `intel_device_info_ehl_4x8`, etc.
- Xe-LP
  - Tiger Lake, 2020, Intel 7 (10nm), 11th gen, Gfx12
    - `intel_device_info_tgl_gt1`, `intel_device_info_rkl_gt05`, etc.
  - Xe (DG1), 2021, Intel 7, Gfx12
    - `intel_device_info_dg1`, etc.
  - Alder Lake, 2021, Intel 7, 12th gen, Gfx12
    - `intel_device_info_adl_gt05`, etc.
  - Raptor Lake, 2022, Intel 7, 13th gen, Gfx12
    - `intel_device_info_rpl`, etc.
- Xe-HPG
  - Arc (DG2, Alchemist), 2022, 6nm, Gfx12.5
    - `intel_device_info_dg2_g10`, etc.
- Xe-LPG
  - Meteor Lake, 2023, Intel 4, 14th gen, Gfx12.5
    - `intel_device_info_mtl_u`, etc.
  - Arrow Lake, 2024, 3nm, Ultra 1st gen, Gfx12.5
    - `intel_device_info_arl_u`, etc.
- Xe2
  - Arc (DG3, Battlemage), 2024, 5nm, Gfx20
    - `intel_device_info_bmg`
  - Lunar Lake, 2024, 3nm, Ultra 2nd gen, Gfx20
    - `intel_device_info_lnl`
- Xe3
  - Arc (DG4, Celestial)
  - Panthor Lake
    - `intel_device_info_ptl`, Gfx30
- Xe4
  - Arc (DG5, Druid)

## Ice Lake

- a GPU has several engines
  - display
  - media
  - GPE, graphics processing engine
- Gfx11LP GT2
  - 1 Unslice, which has
    - 1 GTI
    - 1 Command Streamer
    - 1 Media Fixed Function
    - 1 Blitter
    - 1 Thread Dispatch
    - 1 Geometry
    - 1 POSH
      - tiler
  - 1 slice, which has
    - 1 configurable L3$
      - URB
      - tiler output
      - cache
    - 1 Slice Common
      - 1 Raster
      - 1 HIZ/Z
      - 1 Pixel Dispatch
      - 1 Pixel Backend
    - 8 SubSlices, each has
      - 1 I$ and thread dispatch
      - 1 Sampler
        - Tex$
        - L1$
      - 1 Media Sampler
      - 1 SLM
        - shared local memory
      - 1 Dataport
        - load/store
      - 8 EUs, each has
        - 2 4-wide SIMD ALUs
        - 7 4KB General Register Files, each has
          - 128 GRF registers, each 8x32 bits
          - allow 7 threads
        - 7 Architecture Register Files
        - 1 Thread Control
          - Thread Arbiter picks instructions to run from runnable threads
          - dispatch instructions
          - co-issue up to 4 instructions per cycle
        - 1 Branch unit
        - 1 Send unit

## PRMs

- <https://www.intel.com/content/www/us/en/docs/graphics-for-linux/developer-reference/1-0/overview.html>
  - Hardware Specifications
  - 2023 Alchemist/Arctic Sound-M (Gfx12.5)
    - `intel_device_info_dg2_g10`, `intel_device_info_dg2_g11`,
      `intel_device_info_dg2_g12`, `intel_device_info_atsm_g10`, and
      `intel_device_info_atsm_g11`
  - 2021 Rocket Lake (Gfx12)
    - `intel_device_info_rkl_gt05` and `intel_device_info_rkl_gt1`
  - 2020-2021 Tiger Lake (Gfx12)
    - `intel_device_info_tgl_gt1` and `intel_device_info_tgl_gt2`
  - 2020 Lakefield (Gfx11)
    - `intel_device_info_ehl_4x8`, `intel_device_info_ehl_4x6`,
      `intel_device_info_ehl_4x5`, `intel_device_info_ehl_4x4`,
      `intel_device_info_ehl_2x8`, and `intel_device_info_ehl_2x4`
  - 2020 DG1 (Gfx12)
    - `intel_device_info_dg1` and `intel_device_info_sg1`
  - 2019 Ice Lake (Gfx11)
    - `intel_device_info_icl_gt2`, `intel_device_info_icl_gt1_5`,
      `intel_device_info_icl_gt1`, and `intel_device_info_icl_gt0_5`
    - vol1: preface
    - vol2: command reference
    - vol3: gpu overview
    - vol4: configurations
    - vol5: memory data formats
    - vol6: memory views
    - vol7: memory cache
    - vol8: command stream programming
    - vol9: render engine
    - vol10: copy engine
    - vol11: media engines
    - vol12: display engine
    - vol13: sw/hw system interface
    - vol14: workarounds
  - 2018-2019 Whiskey Lake (Gfx9)
    - same as coffee lake
  - 2017-2019 Amber Lake (Gfx9)
    - same as kaby lake or coffe lake
  - 2017-2019 Coffee Lake (Gfx9)
    - `intel_device_info_cfl_gt1`, `intel_device_info_cfl_gt2`, and
      `intel_device_info_cfl_gt3`
  - 2019-2020 Comet Lake (Gfx9)
    - same as coffee lake
  - 2016 Kaby Lake (Gfx9)
    - `intel_device_info_kbl_gt1`, `intel_device_info_kbl_gt1_5`,
      `intel_device_info_kbl_gt2`, `intel_device_info_kbl_gt3`, and
      `intel_device_info_kbl_gt4`
    - `intel_device_info_glk` and `intel_device_info_glk_2x6`
  - 2016 Apollo Lake and Broxton (Gfx9)
    - `intel_device_info_bxt`, and `intel_device_info_bxt_2x6`
  - 2015-2016 Skylake (Gfx9)
    - `intel_device_info_skl_gt1`, `intel_device_info_skl_gt2`,
      `intel_device_info_skl_gt3`, and `intel_device_info_skl_gt4`
  - 2014 Bay Trail (Gfx7)
    - `intel_device_info_byt`
  - 2014-2015 Cherry Trail and Braswell (Gfx8)
    - `intel_device_info_chv`
    - vol1-vol9: same as broadwell
    - vol10: hevc
    - vol11: blitter
    - vol12: display
    - vol13: mmio
    - vol14: observability
    - vol15: graphics pci registers
    - vol16: workarounds
  - 2014-2015 Broadwell (Gfx8)
    - `intel_device_info_bdw_gt1`, `intel_device_info_bdw_gt2`, and
      `intel_device_info_bdw_gt3`
    - vol1-vol12: same as haswell
    - vol13: mmio
    - vol14: observability
    - vol15: workarounds
  - 2013 Haswell (Gfx7.5)
    - `intel_device_info_hsw_gt1`, `intel_device_info_hsw_gt2`, and
      `intel_device_info_hsw_gt3`
    - vol1: preface and overview
    - vol2: command reference
      - enumerations
      - instructions
      - registers
      - structures
    - vol3: gpu overview
    - vol4: configuration
    - vol5: memory views
      - tiling, gtt, faulting, mocs, formats, surfaces
    - vol6: command stream programming
      - cmd formats, blitter engine, render engine, video command streamer,
        video enhancement command streamer, resource streamer
    - vol7: 3d/media/gpgpu
    - vol8: media vdbox
    - vol9: media vebox
    - vol10: blitter
    - vol11: display
    - vol12: pcie configuration registers
    - observability performance counters
  - 2012 Ivy Bridge (Gfx7)
    - `intel_device_info_ivb_gt1` and `intel_device_info_ivb_gt2`
  - 2011 Sandy Bridge (Gfx6)
    - `intel_device_info_snb_gt1` and `intel_device_info_snb_gt2`
  - 2010 Iron Lake (Gfx5)
    - `intel_device_info_ilk`
    - vol1: graphics core
      - mmio, media regs, env
      - memory interface, render engine
      - video codec engine
      - blitter engine
    - vol2: 3d and media
    - vol3: display registers
      - vga
      - cpu
      - pch
    - vol4: subsystem and cores
      - shared functions
      - message gateway, urb, video motion, and ISA
  - G45 (Gfx4.5)
    - `intel_device_info_g4x`
    - the gpu is in north bridge
  - 965 and G35 (Gfx4)
    - `intel_device_info_i965`
    - the gpu is in north bridge
    - vol1: graphics core
      - this is urb, pipe control, surfaces, memory interface, blt, etc.
    - vol2: 3d/media
      - this is 3D pipeline and media pipeline
    - vol3: display registers
      - this is display
    - vol4: subsystem and cores
      - this is EUs and shared functions
  - `OpRegion` and software `SCI`
    - this is for display
    - the system bios reserves some memory as Operation Regions
    - the kernel driver maps meory to access system states
    - the kernel driver can also `SCI` to call into the bios
- there are 3 engines
  - render engine (vol1 part 3, vol 2, and vol 4)
  - video codec engine (vol1 part 4)
  - blitter engine (vol1 part 5)
- each engine has its own ring buffer and command streamer/parser
- the render engine (or graphics processing engine, GPE) consists of
  - 3D and media "fixed-function" pipelines
  - GEN subsystem, which is used by the pipelines to run kernels
- the video codec engine (VCE) consists of
  - several units to decode and encode videos
- the blitter engine (BLT)
  - is added since SNB

## Tiger Lake, Vol 3, GPU Overview

- the gpu consists of several independent engines
  - display/overlay
    - controlled by mmio directly
  - blitter
    - controlled by mmio directly or MI indirectly
  - GPE (3D, Media, and 3D/Media Subsystem)
    - controlled by MI indirectly
  - VCE
    - controlled by MI indirectly
- the 3D/Media subsystem consists of
  - an array of EUs
  - URB
  - shared functions (sampler, math, itc, cc, render cache)

## Tiger Lake, Vol 4, Configurations

- a slice has 2, 3, 5, or 6 dual subslices, depending on the configurations
- each dual subslice consists of 16 EUs
- each EU has a 8-wide ALU
  - the hw can pair 2 EUs for SIMD16 execution, and that's why it is called a
    dual subslice
- GFLOPS
  - at 1.35GHz, `1.35*96*8=1036.5` GOPS
  - mul takes 1 cycle and thus has 1036.5 GOPS (for both fp32 and int32)
  - mad also takes 1 cycle; it can be said to be 2073 GFLOPS if we consider
    mad as mul+add
  - there is packed math for fp16/int16; it can be said to be 4146 GFLOPS
  - there is DP4A for int8; it can be said to be 8292 GFLOPS

## VUE, thread payload, and 3D pipeline FF stages

- A vertex is stored in URB as a VUE.  An FF stage identifies a vertex by the
  URB handle of the VUE.
- Each vertex also has associated primitive topology information describing
  - `PrimType`, the type of the primitive the vertex belongs to
  - `StartPrim`, true if the vertex is the first one
  - `EndPrim`, true if the vertex is the last one
  - these info are described in thread payload header
- VF unit references `3DSTATE_VERTEX_ELEMENTS` to read
  data from `3DSTATE_VERTEX_BUFFER_STATE` and assemble vertices as VUEs in
  URB.
  - Each vertex element defines 4 continous DWords in VUE.
- VS unit will buffer the VUEs from VF unit, then pass each pair of them
  to a VS thread.
  - the unit will construct the thread payload, which will be preloaded to GRF
    registers for the thread
    - r0: thread payload header (scrach space offset, binding table pointer,
          sampler state pointer, URB return handles, and etc.)
    - rN-rM: constant data (optional) and vertex data
    - for each VUE, `VUE Read Offset` and `VUE Read Length` define the region
      to be read.  They are preloaded to rN (suppose there is no constant
      data) and onward, where N is defined by `Dispatch GRF Start Register`
      and M is `N + VUE Read Length - 1`
  - the thread will output a new VUE for each input VUE, with the new VUE
    defined as
    - D0: MBZ (must be zero)
    - D1: RTAIndex (usually zero)
    - D2: index to `CLIP_VIEWPORT` and `SF_VIEWPORT` array
    - D3: point width
    - D4-7: vertex position
    - D8-15: 8 clip distances (optional)
    - D8- or D16-: arbitrary vertex data (color, texcoord, and etc.)
    - the max size of the arbitrary vertex data is limited by
      `VUE allocation size`, specifed in `3DSTATE_URB`
- GS unit buffers incoming VUEs (from VS unit), assembles them into
  primitives, and passes VUEs of each primitive to a GS thread
  - the unit will construct the thread payload, which will be preloaded to GRF
    registers for the thread
    - r0: thread payload header (scratch, binding table and sampler state
          pointers, edge indicators, `PrimType`, and etc.)
    - r1: SVBI (streamed vertex buffer index)
    - rN-rM: constant data (optional) and vertex data
    - similar to VS thread, except that there can be multiple VUEs preloaded.
      Suppose the first vertex occupies rN and r(N+1), the second vertex
      occupies r(N+2) and r(N+3), and so on.
  - the thread can output new VUEs to URB or to memory buffers via DataPort.
    The latter is for stream output.
    - VUEs are written in write one, allocate another one fashion
- CLIP unit buffers incoming VUEs (from GS unit) and calculates their
  `outCode`s (which clip planes a VUE is considered "outside").  Once a
  complete primitive is received, the primitive is clipped
- SF unit receives inputs from CLIP unit (instead of complete VUEs for
  performance reason?)
  - it will receive
    - per-vertex RTAIndex, VPIndex, point size, and position
      - as in VUE D0 to D7
    - `PrimType`, `PrimStart` and `PrimEnd`
      - as in thread payload header
    - VUE handles
  - its output to WM unit is opaque.  But that includes information needed to
    rasterize the primitives
- WM unit receives opaque inputs from CLIP unit
  - the unit will construct the thread payload which will be preloaded to GRF
    registers for the thread
    - r0: thread payload header (scratch, binding table, sampler, and CC
          pointers, VPIndex, RTAIndex, facing, PrimType, and etc.)
    - r1-2: more payload header fields (sample mask, screen coordinates for
            subspans)
    - r3-26: delivered if the corresponding barycentric interp mode bit is set
             in `WM_STATE`
    - r27-28: delivered if `PS uses source depth` is set in `WM_STATE`
    - r29-30: delivered if `PS uses source W` is set in `WM_STATE`
    - r31: delivered if `Position XY Offset Select` is set in `WM_STATE`
    - r32-55: more barycentric param in 32-pixel dispatch
    - r56-57: more `PS uses source depth` in 32-pixel dispatch
    - r58-59: more `PS uses source W` in 32-pixel dispatch
    - r60: more `Position XY Offset Select` in 32-pixel dispatch
    - rN-M: constant data (optional) and setup data
      - N and M are defined similar to VS or GS
      - setup data are from SF unit.  The amount of data is determined by
        `Number of Output Attributes` field in `3DSTATE_SF`.  Each attribute
        takes two registers.

## Device Programming Environment

- p41, vol 1, part 2
- the graphic device is programmed via three mechanisms
  - POST-time programming of configuration registers
    - not covered here
  - direct (physical i/o and/or mmio) access of graphics registers
  - command stream DMA (via command ring buffer and batch buffers)
    - software writes commands to a command buffer (ring or batch one), and
      uses direct access to inform the device to execute.
- memory-resident commands (or packets) can be categorized into
  - memory interace (MI) commands: control and synchronize the command stream
  - 2D commands: program BLT
  - 3D commands: program 3D pipeline state and perform 3D rendering
  - video decode commands
- commands are written to memory and are read by the graphic device's command
  parser (CP) via DMA
  - it detects the presence of commands (in the ring buffer)
  - it reads commands from the ring buffer and batch buffers
  - it parses the "command type" filed of the commands
  - it executes MI commands
  - it redirects 2D, 3D, and media commands to the appropriate destinations

## GEN Subsystem Overview

- p. 5, vol4, part 1
- The GEN subsystem consists of an array of execution units (EUs) along with a
  set of shared functions.
- Shared functions are hardware units that provide specialized
  functionalities.
- EUs invoke shared functions by sending "messages".  A message is defined by
  a sequential series of MRF (message register file) registers, holding the
  operand, shared function id, and etc.  It also specifies GRF registers that
  the shared function will use for the returned value.
- List of shared functions
  - Extended Math Function
  - Sampling Engine Function
  - DataPort Function
  - Message Gateway Funcion
  - Unified Return Buffer (URB)
  - Thread Spawner (TS)
  - Null Function
- An EU is a vector machine.  It supports multiple execution contexts, called
  threads.  For example, when thread A invokes the slow sampler shared
  function, the EU can switch to thread B and be kept busy.
- Each shared function is identified by a 4-bit function id.
  - It may support one or more operations, called sub-functions.
  - E.g., the extended math shared functions supports sine, cosine and other
    sub-functions.
- DataPort function provides a read/write I/O port.  It is the only way that
  the GPE engine writes results back to memory.  It contains the render and
  depth caches.
- Message Gateway allows a thread to send messages to another thread.  It is
  used by the media pipeline.
- URB is a set of registers that EU threads use to return results to FF
  (fixed-function) units.
- A message consisits of
  - message payload, which is sourced from (1 to 15) MRF registers.
  - associated ("sideband") information about the message, such as target
    function id, payload length, response length, and etc.
- The lifetime of a message has 4 basic phases
  - creation: the thread assembles the message payload by writing to MRF
              registers
  - delivery: the thread executes "send" instruction.  The instruction
    specifies how the message payload are sourced (starting MRF register and
    length), the destination function id, and where in GRF the response is to
    be directed.
  - processing
  - writeback
- When a message is in delivery, attemping to write to an MRF register sourced
  by the message will stall the calling thread.
- Each instruction has an 16-bit execution mask, indicating which channels are
  enabled for the instruction.

## URB

- p. 18, vol4, part 2
- there are 1024 256-bit URB rows in SNB GT1 and 2048 256-bit URB rows in SNB
  GT2
- An URB entry is a logical entity comprised of some number of consecutive row
- Writers
  - VF writes vertex data to URB
  - Threads wrtie data to URB through `URB_WRITE` message
  - there is no CURBE in SNB
- Readers
  - Thread Dispatcher (TD) reads URB rows specified by FF units and provides
    the data as thread payload, which is preloaded to GRF registers
  - GS and CLIP FF units read URB entries to extract vertex data
  - WM FF unit reads depth coefficients from URB entries written by SF FF unit
  - Note that neither CPU nor threads can read URB directly
- FF units snoop `URB_WRITE` messages (for URB readback or passing the entries
  down the pipeline)
- `URB_WRITE` message descriptor (as part of `send` instruction)
  - there is a `Complete` bit allowing partial URB entry writes
  - there is an `Allocate` bit allowing a thread to request another URB entry
  - there is `Swizzle Control` to write all data to a single URB entry, or to
    write each of the upper 128 bits to entry 1 and each of the lower 128 bits
    to entry 0
  - there is `Offset` to offset the writes to URB entry (an entry comprises of
    multiple URB rows, remember?)
- `URB_WRITE` message
  - MRF register `M0` contains the header
  - When writing vertex data without swizzle, `Mx.y` contains
    `VertexData[8 * (x - 1) + y]`
  - When writing interleaved vertex data, `Mx.y` contains
    - `VertexData1[4 * (x - 1) + y]` if `y >= 4`
    - `VertexData0[4 * (x - 1) + y]` if `y <= 3`
- `3DSTATE_VS` specifies "Vertex URB Entry Read Length"
  - Specifies the number of pairs of 128-bit vertex elements to be passed into
    the payload for each vertex.
  - For SIMD4x2 dispatch, each vertex element requires one GRF of payload
    data, therefore the number of GRFs with vertex data will be double the
    value programmed in this field.
  - i965 programs this to `(nr_attributes + 1) / 2`
- `3DSTATE_URB` specifies "VS URB Entry Allocation Size" and "VS Number of URB
  Entries"
  - the former specifies the length of each URB entry owned by VS, in 1024-bit
    units.
  - i965 programs this to `(nr_attributes + 7) / 8`
  - the latter specifies the number of URB entries that are used by VS
  - i965 programs it to `total_urb_size / size_of_an_vs_entry`

## EU ISA and data types

- p. 31 and 35, vol4, part 2
- fundamental types
  - halfbyte
  - byte
  - word
  - doubleword
  - quadword
  - double quadword
  - quad quadword
- they must be naturally aligned in register and memory
- numerical types
  - signed halfbyte integer
  - signed/unsigned byte integer (UB and B)
  - signed/unsigned word integer (UW and W)
  - signed/unsigned doubleword integer (UD and D)
  - packed (32-bit) signed halfbye integer vector (V)
  - restricted 8-bit float
  - 32-bit float (F)
  - packed (32-bit) restricted 8-bit float (VF)
- type conversions
  - float to integer by rounding toward zero
  - integer to float
    - if integer has less than or equal to 24 bits, no rounding
    - otherwise, integer is rounded to nearest event first
  - integer to integer with higher precision
    - unsigned to signed/unsigned by zero extension
    - singed to signed by sign extension
    - singed to unsigned by zero extension
      - with saturation, negative number is saturated to zero first
  - integer to integer with same precision
    - unsigned to signed (saturation matters)
    - signed to unsigned (saturation matters)
  - integer to integer with lower precision
    - bit truncation (saturation matters)

## Execution Environment

- p. 52, vol4, part 2
- In AoS, each verctor is in a register
- In SoA, the same channel of multiple vectors is in a register
- Vertex shaders are usually executed on a base of vertex pairs
  - That is, inputs are arranged in AoS structure and run in SIMD4x2 or SIMD4
    modes
- Fragment shaders are usually executed on a base of 16-pixel groups
  - That is, inputs are arranged in SoA structure and run in SIMD8 or SIMD16
    modes
- Shared functions have transpose to suppor both SoA and AoS.
- SIMD4, SIMD4x2, SIMD8, SIMD16
  - see section 5.2
  - SIMD8 is a special case of SIMD16.  It is used when arthitectural
    restrictions apply.  For example, the address register a0 has only 8
    elements.  If indirect addressing is used, SIMD16 instruction is not
    allowed.
- An EU has 640 GRF registers.  Each thread is assigned 128 of them, starting
  from r0 to r127.  Each GRF register is 256-bit wide.
- Each thread is assigned 24 MRF registers.  Each MRF register is 256-bit
  wide.
- ARF registers
  - there is one Null register
  - there is one Address register, a0.  It can contain 8 UW.  Its size is
    128-bit wide.
  - there are two Accumulator registers, acc0 and acc1.  It can contain 8
    D or 16 W.
  - there is one Flag register, f0.  It contains two UW.
  - there is one State register, sr0, containing 4 UD.
  - there is one Control register, cr0, containing 4 UD.
    - cr0.2:ud contains AIP, the IP before exception
  - there is one Notification register, cr0, containing 3 UD.
  - there is one IP register, ip0, containing 1 UD.
  - there is one TDR (thread dependency) register, tdr0, containing 8 UW.
- Immediates can be scalars or vectors
  - scalar immediates must be words or doublwords (not bytes!)
  - the immediate field in an instruction has 32 bits.  When it is a word, its
    bits must be duplicated in the higher bits
  - vector immediates must be V or VF
- Source operand region parameters
  `|RegFile||RegNum|.|SubRegNum|<|VertStride|;|Width|,|HorzStride|>:|Type|`
  - RegNum is in unit of 256-bit.  SubRegNum is in unit of the size of Type
  - Width specifies the number of elements in a row.
  - HorzStride specifies the step size between two adjacent elements in a row,
    in unit of the size of Type
  - VertStride specifies the step size between two adjacent elements along the
    vertical dimension.  It is again in unit of the size of Type.
- Destination operand region parameters
  `|RegFile||RegNum|.|SubRegNum|<|HorzStride|>:|Type|`
  - it is one-dimentional.
  - in other words, source operands may span multiple physical registers,
    while destination operand must be contained in a physical register.
- The number of elements in a region is called the size of the region.
- execution size (ExecSize), specified in the instruction word, determines the
  size of the region
  - For source operand, Height can be derived from `ExecSize / Width`
- address registers can be used to provide RegNum.SubRegNum (1x1) or
  multiple RegNum.SubRegNum (VxH)
- Access Modes
  - `align16` has 16-byte alignment requirements, but is full featured.
    Suitable for AoS.
  - `align1` has no alignment requirements, but does not support source
    swizzle or destination mask.  Suitable for SoA.
- Instructions can have operands of different types.  The types of the source
- operands determine the execution type of the instruction.  All source
  operands are converted to the type for computation.  When the dst operand
  has a different type from the execution type, the result is converted again
  before being written to the dst register.
  - the rules are
    - integers and floats cannot be mixed
    - if any source operand is dword, the execution type is signed D
    - otherwise, the execution type is signed W
- Predication

## Instruction Set Summary

- p. 127, vol4, part 2
- An instruction work has 128 bits
- Common fields
  - `CondModifier`: 4 bits
    - sets the flag register
  - `AddrMode`: 1 bit
    - determine whether an operand is directly or indirectly addressed
  - `RegNum`: 8 bits
    - For GRF or MRF operands, this specifies the register address in unit of
      256-bits
    - For ARF operand, higher 4 bits specify the register type; lower 4 bits
      specify the register number
  - `SubRegNum`: 5 bits
    - For GRF or MRF operands, it specifies the byte address within the
      256-bit register
    - For ARF operand, it specifies the byte address within the ARF register
  - `AddrSubRegNum`: 3 bits
    - it specifies which of the eight address subregisters (a0.#) is used to
      indirectly address the register
    - it is used in register-addressing mode.  `RegNum` and `SubRegNum` are
      ignored.
  - `AddrImm`: 10 bits
    - an signed offset added to the address register to compute the final
      register address
    - used only in register-addressing mode
  - `SrcMod`: 2 bits
    - negate and/or absolute the source operand
  - `VertStride`: 4 bits
    - 0 means zero element; n (from 1 to 6) means 2^(n -1) elements.  Only
      applies to source operands.
    - for `align16` mode, only zero or 4 elements are allowed
    - 0xf means VxH or Vx1 indirect addressing
  - `Width`: 3 bits
    - number of elements, in power of 2, in the horizontal dimension
  - `HorzStride`: 2 bits
    - the distance between to adjacent elements in a row.  same meaning as
      `VertStride`
  - `Imm32`: 32 bits
    - immediate of scalar, V, or VF types
  - `ChanEn`: 4 bits
    - write mask, only applies to dst operand in `align16` mode
    - if a channel is enabled, ch, ch+4, ch+8, ... are enabled if `ExecSize`
      is greater than 4.
  - `ChanSel`: 8 bits
    - source swizzle
  - `MsgDscpt31`: 31 bits
    - message descriptor, only available to `send`
  - `EOT`: 1 bit
    - end-of-thread, only available to `send`
- DW0: instruction operation doubleword
  - `Saturate`: 1 bit
  - `CondModifier`: 4 bits
    - `CurrDst.RegNum` if `send` instruction
  - `ExecSize`: 3 bits
  - `PredCtrl`: 4 bits
  - `AccessMode`: 1 bit (align16 or align1)
  - `Opcode`: 7 bits
- DW1: instruction destination doubleword
  - Src1 `SrcType`
  - Src1 `RegFile`
  - Src0 `SrcType`
  - Src0 `RegFile`
  - Dst `DstType`
  - Dst `RegFile`
  - Dst Region
    - `AddrMode`
    - `RegNum`
    - `SubRegNum`
    - `ChanEn`
    - `HorzStride` if align1
    - `AddrSubRegNum` if align1
    - `AddrImm` if align1
- DW2: instruction source-0 doubleword
  - `FlagSubRegNum`
  - `VertStride`
  - `ChanSel`
  - `AddrMode`
  - `SrcMod`
  - `RegNum`
  - `SubRegNum`
  - `Width` if align1
  - `HorzStride` if align1
  - `AddrSubRegNum` if align1
  - `AddrImm` if align1
- DW3: instruction source-1 doubleword
  - similar to DW2
  - or `Imm32`

## `send` instruction

- `send <dst> <src> <ex_desc> <desc>` sends a message stored in MRF starting
  at `<src>` to a shared function identified by `<ex_desc>` along with control
  from `<desc>` and GRF writebacks starting at `<dest>`
- `<ex_desc>` is a 6-bit immediate, with the higest bit being EOT and the
  lowest 4 bits being shared function id
- `<desc>` is a 32-bit immediate
  - bit 25..28: message length (number of MRF registers starting from src)
  - bit 20..24: response length (number of GRF registers starting from dst)
  - bit 19: header present
    - for URB, this must be 1
  - bit 0..18: function control
    - for URB
      - bit 15: complete
      - bit 14: used
      - bit 13: allocate
      - bit 10..11: swizzle control
      - bit 4..9: offset
      - bit 0..3: URB Opcode
- for `URB_WRITE`, src is the header
  - m0.0: handle id and urb return handle 0
  - m0.1: handle id and urb return handle 1
  - ...

## flag register, condition modifier, and predicate

- if `CondMod` is specified, the per-channel results are checked and flag
  register is updated
- flag register is used to generate `PMask` for the next instruction,
- When an instruction is executed, `WrEn` is evaluated
  - if `WeCtrl` is on, all first `ExecSize` bits are set
  - else, ???
  - if `PredCtrl`, `WrEn` is masked by `PMask`
- Some instructions are special
  - `sel`
  - `cmp`
  - `if` and family

## VS unit

- When the VS unit spawns a thread in SIMD4x2 mode, it will pass these as the
  thread payload
  - R0: header providing pointers to VS binding table, VS sampler states, the
    URB return offsets.  The thread should output the processed vertices to
    the URB return offsets.
  - optional constant data: the length is specified by `3DSTATE_CONSTANT_VS`
  - vertex data: the length is specified by "Vertex URB Entry Read Length".
                 The lower 128-bit of each GRF register always contains data
                 from one VUE.  The higher 128-bit may contain data from
                 another VUE.
