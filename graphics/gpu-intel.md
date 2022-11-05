# Intel GPUs

## uArchs

- Nehalem
  - Nehalem, 2008, 45nm, 1st gen, Gen4
  - Westmere, 2010, 32nm, 1st gen, Gen5
- Sandy Bridge
  - Sandy Bridge, 2011, 32nm, 2nd gen, Gen6
  - Ivy Bridge, 2012, 22nm, 3th gen, Gen7
- Haswell
  - Haswell, 2013, 22nm, 4th gen, Gen7.5
  - Broadwell, 2014, 14nm, 5th gen, Gen8
- Skylake
  - Skylake, 2015, 14nm, 6th gen, Gen9
  - Kaby Lake, 2016, 14nm, 7th gen, Gen9.5
  - Coffee Lake, 2017, 14nm, 8th/9th gen, Gen9.5
  - (SKIPPED) Cannon Lake, 2018, 10nm, 9th gen, Gen10
  - Comet Lake, 2019, 14nm, 10th gen, Gen9.5
- Ice Lake
  - Ice Lake, 2019, 10nm, 10th gen, Gen11
- Xe
  - Tiger Lake, 2020, Intel 7 (10nm), 11th gen, Gen12
  - Alder Lake, 2021, Intel 7 (10nm), 12th gen, Gen12.5
  - Raptor Lake, 2022, Intel 7 (10nm), 13th gen, Gen12.5
  - DG1, 2020
  - Arc (DG2), 2022

## Ice Lake

- a GPU has several engines
  - display
  - media
  - GPE, graphics processing engine
- Gen11LP GT2
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

## IRQs

- `gen8_de_irq_handler` for display engine related interrupts
  - `GEN8_PIPE_VBLANK` is generated on vblank, if anyone holds on to vblank
    with `drm_vblank_get`
- `gen8_gt_irq_handler` for graphics related interrupts
  - `GT_RENDER_USER_INTERRUPT` is generated in response to `MI_USER_INTERRUPT` cmd
  - `GT_CONTEXT_SWITCH_INTERRUPT` is generated on context switch
    - always on
    - programmed to be generated at the end of each request
    - requests are submitted to the HW on this interrupt.
    - when GPU is idle, it runs the idle context.  A single request causes
      two context switches: idle->req->idle

## Swap Buffers

### Non-fullscreen copying

- app flush
  - `I915_GEM_EXECBUFFER2`
    - event `dma_fence_init`
      - this execbuffer allocates a `i915_request`, which is associated with a
      	`dma_fence`.
    - event `i915_request_queue`
      - we are about to queue the request to the HW ring buffer
    - event `i915_request_add`
      - i915 starts tracking the request status
  - request retiring
    - i915 occasionally checks the status of active requests and retires them
    - when a request is waited for, signaling of the `dma_fence` is enabled
      (i.e., IRQ is enabled) for timley retiring
    - either way, it generates a `dma_fence_signaled` event
- app swap buffers
- X waits for vsync
  - event `drm_vblank_event`
    - this indicates a vblank
- X copies to the window frontbuffer (which is a region of the scanout buffer)
  - `I915_GEM_EXECBUFFER2` to copy
  - `DRM_IOCTL_MODE_DIRTYFB` to dirty the region
    - clflush
  - if GPU copy runs too long, or if GPP rendering for the app takes too long,
    tearing can be seen
- Latency: minimal (presented next vsync after swap buffers)

### Fullscreen flipping

- app swap buffers
- X modesetting driver flips
  - `DRM_IOCTL_MODE_ADDFB2`
  - `DRM_IOCTL_MODE_ATOMIC` -> non-blocking `intel_atomic_commit`
    - `intel_atomic_prepare_commit`
      - calls `intel_prepare_plane_fb` and adds the BO implicit fence that
      	will be waited for
      	* the first time a BO is used for scanout, `intel_plane_pin_fb` calls
      	  `i915_gem_object_pin_to_display_plane` which calls
      	  `i915_gem_object_set_cache_level` which calls
      	  `i915_gem_object_wait`.  The commit becomes blocking.
      	* also in `i915_gem_object_pin_to_display_plane` for the first time,
      	  `__i915_gem_object_flush_for_display` clflushes and adds a fence
      	* they are gone after the first use
    - `drm_atomic_helper_setup_commit` makes sure the previous flip has
      completed, otherwise returns EBUSY
  - `intel_atomic_commit_tail` is called in a wq because of non-blocking
    - it waits the BO first in `intel_atomic_commit_fence_wait`
      - because of the issue in `intel_plane_pin_fb`, it waits for `clflush`
      	worker
    - it waits for the previous flip to complete
      - no-op because of `drm_atomic_helper_setup_commit`
    - it writes to the double-buffered registers
      - event `i915_pipe_update_start`
      - event `intel_update_plane`
      - event `i915_pipe_update_end`
    - it waits for the current flip to complete
      - sends `DRM_EVENT_FLIP_COMPLETE`
      - event `drm_vblank_event`
- Latency: minimal (presented next vsync after swap buffers)
 
### Composited

- app swap buffers
- X waits for vsync
- X wakes up the compositor
- X copies to the window frontbuffer (which is a fake frontbuffer)
  - X copies on vsync even for fake frontbuffer?  This is unnecessary, unless
    it wants to throttle the compositor
- compositor, which is an fullscreen client, swap buffers
  - compositor does not want swap buffers whenever any client has a new buffer
  - it runs on vsyncs; if any client has a new buffer, it recomposites
    everything and swap buffers
  - but there are compositors that simply swap buffers whenever there is any
    damage
- X modesetting driver flips
  - or copies, if the compositor does server-side rendering
- Latency: +one vsync (presented next next vsync after app swap buffers)

### virtio-gpu

- guest app swap bufers
- guest X waits for vsync (no-op because virtio-gpu does not support vsync)
- guest X copies to the scanout buffer and dirty the scanout buffer
  - copy uses glamor, which is GL
  - dirty calls `virtio_gpu_framebuffer_surface_dirty`, which emits a
    `VIRTIO_GPU_CMD_RESOURCE_FLUSH`.
- the hypervisor blits from the guest scanout buffer (which is a GL texture on
  host) to the host window, and does a swap buffer.  The hypervisor is a
  regular client to host X.
- Latency: host swap is minimal (presented next vsync after swap buffers);
  From guest swap to host swap, there are two additional copies

## Caching

- `i915_gem_object_pin_map` is called when the kernel needs to access the
  BO from CPU
  - e.g., cmd parsing, reloc patching
  - it calls vmap internally to set up a WB or WC mapping
  - this does not modify the kernel linear map
- `i915_gem_mmap_ioctl` is called when ther userspace needs to access the BO
  from CPU
  - it calls `vm_mmap` to set up a vma for the shmem file, and modifies
    `vma->vm_page_prot` to use the desired cache mode

## Implicit Fencing

- in anv, an `anv_bo` is allocated by `anv_device_alloc_bo`
- all VkDeviceMemory are tracked globally
  - `anv_AllocateMemory` adds them to `dev->memory_objects`
- in `anv_queue_execbuf_locked`, `anv_execbuf` is initialized from
  `anv_queue_submit`
  - `anv_execbuf_add_bo` is used to add an `anv_bo` to `anv_execbuf`
  - all but wsi bos are added without `EXEC_OBJECT_WRITE`
  - in `setup_execbuf_for_cmd_buffer`, all internally tracked bos and all
    VkDeviceMemory bos are added to `anv_execbuf`
- in kernel `i915_gem_do_execbuffer`,
  - `eb_lookup_vmas` makes sure each `drm_i915_gem_object` has a `i915_vma`
  - `eb_relocate_parse` calls `eb_validate_vmas`, which calls
    `eb_pin_vma` to pin bo pages inside `i915_vma_pin_ww`
  - `eb_submit` calls `eb_move_to_gpu`, where `i915_vma_move_to_active` calls
    `dma_resv_add_excl_fence` or `dma_resv_add_shared_fence` depending on
    whether `EXEC_OBJECT_WRITE` is set

## ioctls of a simple vk frame

- acquire the frame image
  - `anv_AcquireNextImageKHR`
    - no ioctl
- make sure the frame is "idle" because we reuse frame objects
  - `anv_WaitForFences`
    - `DRM_IOCTL_SYNCOBJ_WAIT`
  - `anv_ResetFences`
    - `DRM_IOCTL_SYNCOBJ_RESET`
- submit the command buffer
  - `anv_QueueSubmit`
    - `DRM_IOCTL_I915_GET_RESET_STATS` to make sure GPU is healthy
    - `DRM_IOCTL_SYNCOBJ_RESET` to reset `pSignalSemaphores` and `fence`
      - for reasons
    - `DRM_IOCTL_I915_GEM_EXECBUFFER2`
- present the frame
  - `anv_QueuePresentKHR`, which calls `wsi_common_queue_present`
    - `anv_WaitForFences` because common wsi reuses frame objects
    - `anv_ResetFences`
    - `anv_QueueSubmit` which is an empty submit with `pWaitSemaphores` from
      `VkPresentInfoKHR` and fence from common wsi

## Images

- `isl_drm_modifier_info`
  - describes relations between `I915_FORMAT_MOD_*`, `ISL_TILING_*`,
    `ISL_AUX_USAGE_*`
- `anv_format`
  - describes relations between `VK_FORMAT_*`, `ISL_FORMAT_` and isl planes
  - most vk formats have 1 isl plane
  - `VK_FORMAT_D24_UNORM_S8_UINT` has `ISL_FORMAT_R24_UNORM_X8_TYPELESS` and
    `ISL_FORMAT_R8_UINT` planes
  - `VK_FORMAT_*_nPLANE_*` has `n` isl planes
- `add_all_surfaces_implicit_layout` loops through all aspects
  - `VK_IMAGE_ASPECT_*` decides the isl plane index and `anv_format_plane`
  - `VK_IMAGE_ASPECT_*` and `VK_IMAGE_USAGE_*` decide `ISL_SURF_USAGE_*`
  - `add_primary_surface`
    - this initializes `isl_surf` for the main surface state(s)
    - `image_binding_grow` calculates the offset/size of the data in the bo
  - `add_aux_surface_if_supported`
  - after this function, we know the surface states of the image and the
    (offset, size) of each surface which can be used to calculate the memory
    requirements

## PRM

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

## `brw_old_vs_emit`

- static register allocation
  - `r0` is reserved
  - for each pair of user clip planes, a GRF register is allocated
  - for each pair of constants, a GRF register is allocated
  - for each input attribute, a GRF register is allocated
  - for each output varying, a GRF or MRF register is allocated
  - for each temporary, a GRF register is allocated
  - for each indirect addressing reg, a GRF register is allocated
  - 4 GRF registers are allocated for const buffer
  - for each output varying used as a source operand, a GRF register is
    allocated
- a source operand of an instruction may
  - be a constant
    - use CURBE if there are enough space and it is directly addressed
    - send `GEN6_SFID_DATAPORT_SAMPLER_CACHE` to read from constant buffer
    - send the same message with relative addressing if relatively addressed
  - be an input or ouptut
    - use the assigned register if directly addressed
    - otherwise, dereference using the indirect register
- a destination operand of an instruction may
  - be used as a source in another instruction
    - use additionally allocated register as the destination
  - be directly addressed
    - use the allocated register
  - be indirectly addressed
    - use a temp register
