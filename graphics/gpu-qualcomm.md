Qualcomm Adreno
===============

## History

- Snapdragon S series
  - discontinued
  - S1
    - 2007-2008
    - MSM7x27
    - 65nm
    - Adreno 200, GLES 2.0
  - S2
    - 2010
    - MSM7x30, MSM8x55
    - 45nm
    - Adreno 205, GLES 2.0
  - S3
    - 2010
    - MSM8x60
    - 45nm
    - Adreno 220, GLES 2.0
  - S4
    - 2012
    - MSM8x{25,27,30,60}
    - 28nm-45nm
    - Adreno 203, 225, 305, 320
- Snapdragon 2 series
  - ultra budget
  - 200
    - 2013
    - MSM8x{10,12}
    - 28nm
    - Adreno 302
  - 205, 208, 210, 212
    - 2014-2017
    - MSM8x{05,08,09}
    - 28nm
    - Adreno 304
  - 215
    - 2019
    - QM215
    - 28nm
    - Adreno 308
- Snapdragon 4 series
  - entry level
  - 400
    - 2013
    - MSM8x{26,28,30}
    - 28nm
    - Adreno 305
  - 410, 412, 415
    - 2014-2015
    - MSM8x{16,29}
    - 28nm
    - Adreno 306, 405
  - 425, 427, 430, 435
    - 2015-2016
    - MSM8x{17,20,37,40}
    - 28nm
    - Adreno 308, 505
  - 429, 439, 450
    - 2017-2018
    - SDM{429,439,450}
    - 12nm-14nm
    - Adreno 504, 505, 506
  - 460
    - 2020
    - SM4250
    - 11nm
    - Adreno 610
  - 480
    - 2021
    - SM4350
    - 8nm
    - Adreno 619
- Snapdragon 6 series
  - mid-range
  - 600
    - 2013
    - APQ8064
    - 28nm
    - Adreno 320
  - 610, 615, 616
    - 2014-2015
    - MSM89{36,39}
    - 28nm
    - Adreno 405
  - 625, 626
    - 2016
    - MSM8953
    - 14nm
    - Adreno 506
  - 650, 652, 653
    - 2016
    - MSM89{56,76}
    - 28nm
    - Adreno 510
  - 630, 636, 660
    - 2017
    - SDM{630,636,660}
    - 14nm
    - Adreno 508, 509, 512
  - 632, 670
    - 2018
    - SDM{632,670}
    - 10nm-14nm
    - Adreno 506, 615
  - 662, 665, 675, 678
    - 2019-2020
    - SM61{15,25,50}
    - 11nm
    - Adreno 610, 612
  - 680, 690, 695
    - 2020-2021
    - SM{6225,6350,6375}
    - 6nm-8nm
    - Adreno 610, 619
- Snapdragon 7 series
  - upper mid-range
  - 710, 712
    - 2018-2019
    - SDM{710,712}
    - 10nm
    - Adreno 616
  - 720, 730, 732
    - 2019-2020
    - SM71{25,50}
    - 8nm
    - Adreno 618
  - 750, 765, 768
    - 2020
    - SM72{25,50}
    - 7nm
    - Adreno 620
  - 778, 780
    - 2021
    - SM73{25,50}
    - 5nm-6nm
    - Adreno 642
- Snapdragon 8 series
  - high-end
  - 800, 801, 805
    - 2013-2014
    - MSM8x74, APQ8084
    - 28nm
    - Adreno 330, 420
  - 808, 810
    - 2015
    - MSM9x{92,94}
    - 20nm
    - Adreno 418, 430
  - 820, 821
    - 2016
    - MSM8996
    - 14nm
    - Adreno 530
  - 835
    - 2017
    - MSM8998
    - 10nm
    - Adreno 540
  - 845
    - 2018
    - SDM845
    - 10nm
    - Adreno 630
  - 855, 860
    - 2019, 2021
    - SM8150
    - 7nm
    - Adreno 640
  - 865, 870
    - 2020-2021
    - SM8250
    - 7nm
    - Adreno 650
  - 888
    - 2021
    - SM8350
    - 5nm
    - Adreno 660
  - 8 Gen1
    - 2022
    - SM8450
    - 4nm
    - Adreno 730
- Snapdragon 7 Compute Platforms
  - 7c
    - 2020
    - SC7180
    - 8nm
    - Adreno 618
  - 7c Gen2
    - 2021
    - SC7180P
    - 8nm
    - Adreno 618
  - 7c+ Gen3
    - 2021
    - SC7280?
    - 6nm
    - Adreno 7c+ Gen3 (Adreno 635)
- Snapdragon 8 Compute Platforms
  - 835, 850
    - 2018
    - MSM8998, SDM850
    - 10nm
    - Adreno 540, 630
  - 8c, 8cx, 8cx Gen2
    - 2019-2020
    - SC8180
    - 7nm
    - Adreno 675, 680, 690
  - 8cx Gen3
    - 2022
    - SC8???
    - 5nm
    - Adreno 8cx Gen3

## Architecture

- `Qualcomm® Snapdragon™ Mobile Platform OpenCL General Programming and Optimization`
- Adreno
  - a big L2 sitting in from of system memory
    - all memory accesses from SP are via L2
  - a Texture Processor / L1 for Sampling and Image Read
  - multiple Shader Processors
- a Shader Processor (SP) is
  - core block of Adreno GPUs with many moudles including ALU, load/store
    unit, control flow unit, register files, etc.
  - each SP corresponds to a OpenCL compute unit
  - load/store through L2 for buffer objects and (r/w) image objects
  - load/sample through  texture processor / L1 for read-ony image objects
- a Texture Processor (TP) is
  - texture fetching and filtering
  - coupled with L1 which fetches data from L2
- A unified L2 cache (UCHE)
  - respond to SP's load/store and L1's load requests
- Waves and fibers
  - the smallest unit of execution is a fiber (thread...)
  - a collection of fibers execute in lock-step is a wave
  - wave size can be 8, 16, 32, 64, 128, etc.
  - a SP can have multiple active waves, for latency hiding
    - maximum number of active waves depend on register file size and register
      footprint of the waves
  - an OpenCL kernel launches multiple workgroups
    - each workgroup is assigned to an SP
    - each SP processes one (or multiple on highend) workgroup at a time
    - the remaining SPs are queued in GPU for execution
  - an OpenCL workgroup is processed by multiple waves
    - the larger the better, but not always
    - larger workgroups means more waves and better latency hiding
- OpenCL Memory Model
  - global memory is system memory
  - local memory is on-chip GMEM in SP
  - private memory is registers, on-chip GMEM, or system memory decided by
    compiler
  - constant memory is on-chip if can fit; system memory otherwise
  - not coherent between CPU/GPU on A5xx
- Graphics Fixed-Function Pipeline
  - Command Processor (CP)
    - uncached
  - Color Cache Unit (CCU)
    - draws and blits hit CCU

## PM4 Command Packets

- From Radeon South Island Programming Guide
  - Packet DW0[31:30] spcifies the type
  - Type 0 updates registers in the first 64K DW
    - use type 3 instead
    - `type0(u16 startReg, u14 count, u32 value[count])`
  - Type 2 is no-op, to fill up the trailing space
  - Type 3 executes the specified opcode
    - `tyep3(u8 opcode, bool predicate, u32 payload[])`
    - Initialization Packets
      - `MI_INITIALIZE`
    - Command Buffer Packets
      - `INDIRECT_BUFFER`
    - Draw/Dispatch Packets
      - `DRAW_INDEX`
      - `DRAW_INDEX_AUTO`
      - `DRAW_INDIRECT`
    - State Management Packets
    - Command Predication Packets
    - Synchronization Packets
      - `EVENT_WRITE`: initiate an event and optionally write a value to the
        specified address when the event completes
    - Atomic
    - Misc Packets
- From freedreno
  - type 7 works similar to type 3
  - type 4 works similar to type 0

## CP, Command Processor

- CP has ME (microengine) and PFP (prefetch parser)
  - PFP adds commands to a FIFO named MEQ
  - ME processes commands from MEQ
  - since a6xx, they are replaced by SQE
  - see also
    - `afuc/README.rst`
    - `registers/adreno/adreno_pm4.xml`
- command processing
  - `CP_INDIRECT_BUFFER` calls an indirect buffer (IB)
    - actually, CP executes commands from a ring buffer (RB) controlled by
      kernel
    - userspace submissions are IB1 added to RB
    - userspace indrect buffers are IB2 added to IB1
  - `CP_COND_EXEC` conditionally execs the following N dwoard based on values
    at addrs
    - `addr1 != 0`
    - `addr1 < ref`
  - `CP_COND_REG_EXEC` conditionally execs the following N dwords based on reg
    - can be used with `CP_REG_TEST`
    - can check reg1==reg2
    - can test the operation mode (binning, sysmem, or gmem) set by
      `CP_SET_MARKER`
  - `CP_SET_MARKER` tells CP the current operation mode
  - `CP_REG_TEST` tests a reg, as cond exec predicate
- synchronization
  - `CP_WAIT_FOR_ME` tells PFP to wait for ME
  - `CP_WAIT_FOR_IDLE` tells ME to wait for the pipeline to be idle?
  - `CP_WAIT_MEM_WRITES` waits mem writes from ME?
- memory access
  - CP memory access is uncached
  - `CP_MEM_WRITE` writes a dword or qword to addr
  - `CP_MEM_TO_MEM` copies a qword with alu
  - `CP_MEMCPY` copies N dwords
  - `CP_COND_WRITE5` writes addr if val at another addr meets the condition
  - `CP_REG_TO_MEM` copies a reg to addr
  - `CP_WAIT_REG_MEM` waits for addr to meet a condition
- register access
  - `CP_REG_WRITE` writes a reg
    - removed on latest gens
  - `CP_CONTEXT_REG_BUNCH` writes regs
  - `CP_REG_RMW` increments/decrements/rmw a reg
  - `CP_MEM_TO_REG` copies addr to reg
- 2d blit
  - `CP_BLIT` works great for clears, copies, and blits
    - when its restrictions are satisfied
    - non-msaa, supported formats, etc.
- compute dispatch
  - `CP_EXEC_CS` dispaches
  - `CP_EXEC_CS_INDIRECT` dispaches with params on a buffer
- draw / dispatch
  - `CP_SET_MODE` executes `CP_SET_DRAW_STATE` immediately when 0x1
  - `CP_SET_BIN_DATA5_OFFSET` sets binning configuration
  - `CP_SET_VISIBILITY_OVERRIDE` ignores visibility stream when 0x1
  - `CP_SKIP_IB2_ENABLE_GLOBAL` ignores `CP_INDIRECT_BUFFER`
    - can be useful in binning pass
  - `CP_LOAD_STATE6_GEOM` preloads consts, ubos for VS/HS/DS/GS
  - `CP_LOAD_STATE6_FRAG` preloads consts, ubos for FS/CS
  - `CP_SET_DRAW_STATE` is `CP_INDIRECT_BUFFER` on steroids
    - it allows drivers to construct stateobjs (IBs)
    - it can conditionally exec stateobjs depending on the operation mode and
      whether the stateobjs have changed
  - `CP_SET_SUBDRAW_SIZE` sets the tess factor/param buffer size for HS
  - `CP_DRAW_INDX_OFFSET` draws
  - `CP_DRAW_INDIRECT_MULTI` draws indirectly
  - `CP_DRAW_AUTO` draws using transform feedback data
  - `CP_DRAW_PRED_ENABLE_GLOBAL` enables/disables conditional rendering
  - `CP_DRAW_PRED_ENABLE_LOCAL` enables/disables conditional rendering
    internally for internal draws
  - `CP_DRAW_PRED_SET` sets the predicate bit for conditional rendering
- `CP_EVENT_WRITE` generates an "event" and optionally writes to an addr when
  the event completes
  - `BLIT`
  - `CACHE_FLUSH_TS` flushes UCHE
  - `CACHE_INVALIDATE` invalidates UCHE
  - `FLUSH_SO_0(i)` writes SO results to `VPC_SO_FLUSH_BASE(i)`
  - `LRZ_FLUSH` is generated at tile render begin/end and sysmem begin/end
  - `PC_CCU_FLUSH_COLOR_TS` flushes CCU color cache
  - `PC_CCU_FLUSH_DEPTH_TS` flushes CCU depth cache
  - `PC_CCU_INVALIDATE_COLOR` invalidates CCU color cache
  - `PC_CCU_INVALIDATE_DEPTH` invalidates CCU depth cache
  - `PC_CCU_RESOLVE_TS` is generated at tile render end
  - `RB_DONE_TS` waits for end-of-pipe to complete
  - `RST_PIX_CNT` resets pixel count?
  - `RST_VTX_CNT` resets vertex count?
  - `START_PRIMITIVE_CTRS` starts primitive counters
  - `STOP_PRIMITIVE_CTRS` stops primitive counters
  - `STAT_EVENT` for queries
  - `TILE_FLUSH` for queries
  - `UNK_2C`
  - `UNK_2D`
  - `WRITE_PRIMITIVE_COUNTS` writes primitive counts to addr specified in
    `VPC_SO_STREAM_COUNTS` for xfb
  - `ZPASS_DONE` writes zpass fragment count to addr specified in
    `A6XX_RB_SAMPLE_COUNT_ADDR`

## VFD, vertex fetch and decode

- `VFD_CONTROL_[0-6]`
  - how many attrs to fetch/decode
  - which sysvals such as vertex ids to generate
- `VFD_MODE_CNTL`: normal or binning pass
- `VFD_MULTIVIEW_CNTL`: multiview
- `VFD_ADD_OFFSET`: offsets added to sysvals
- `VFD_INDEX_OFFSET`: ofsset to draw firstVertex
- `VFD_INSTANCE_START_OFFSET`: ofsset to draw firstInstance
- `VFD_FETCH_BASE(i)`: vb addr
- `VFD_FETCH_SIZE(i)`: vb size
- `VFD_FETCH_STRIDE(i)`: vert stride
- `VFD_DECODE_INSTR(i)`: vert format, etc.
- `VFD_DECODE_STEP_RATE(i)`: instancing step rate
- `VFD_DEST_CNTL_INSTR(i)`: maps attrs to regs
- `VFD_POWER_CNTL`: magic value

## VSC, visibility stream compressor?

- <https://github.com/freedreno/freedreno/wiki/Visibility-Stream-Format>
  - the draw stream consists of bitmasks, where each bitmask indicates which
    bins are covered by a draw command
    - CP uses it to skip empty draws
  - each primitive stream consists of bitmasks, where each bitmask indicates
    which bins are covered by a primitive
- `VSC_DRAW_STRM_ADDRESS`: draw stream addr
- `VSC_DRAW_STRM_PITCH`: draw stream pitch
- `VSC_DRAW_STRM_LIMIT`: draw stream limit
- `VSC_DRAW_STRM_SIZE_ADDRESS`: draw steam what?
- `VSC_PRIM_STRM_ADDRESS`: prim stream addr
- `VSC_PRIM_STRM_PITCH`: prim stream pitch
- `VSC_PRIM_STRM_LIMIT`: prim stream limit
- `VSC_BIN_SIZE`: tile width and tile height
- `VSC_BIN_COUNT`: tile count in both dims
- `VSC_PIPE_CONFIG(i)`: which pipes handles which tiles
- `CP_SET_BIN_DATA5_*`: slot of the selected tile

## PC, primitive control?

- `PC_TESS_NUM_VERTEX`: for hs
- `PC_HS_INPUT_SIZE`: for hs
- `PC_TESS_CNTL`: for hs
- `PC_RESTART_INDEX`: prim restart index
- `PC_MODE_CNTL`: magic
- `PC_POWER_CNTL`: magic
- `PC_PRIMID_PASSTHRU`: passes through primid
- `PC_POLYGON_MODE`: polygon mode
- `PC_RASTER_CNTL`: rasterizer discard
- `PC_PRIMITIVE_CNTL_0`: prim restart, provoking vertex
- `PC_VS_OUT_CNTL`: regids of pointsize, view, layer, primid, etc.
- `PC_GS_OUT_CNTL`: same but for gs
- `PC_HS_OUT_CNTL`: same but for hs
- `PC_DS_OUT_CNTL`: same but for ds
- `PC_PRIMITIVE_CNTL_5`: for gs
- `PC_PRIMITIVE_CNTL_6`: for gs
- `PC_MULTIVIEW_CNTL`: multiview enable and count
- `PC_MULTIVIEW_MASK`: multiview mask
- `PC_TESSFACTOR_ADDR`: addr of tess factor bo

## VPC, vertex/primitive clip?

- `VPC_GS_PARAM`: where are line length written to
- `VPC_VS_CLIP_CNTL`: where are clip distances written to
- `VPC_GS_CLIP_CNTL`: same but for gs
- `VPC_DS_CLIP_CNTL`: same but for ds
- `VPC_VS_LAYER_CNTL` where are layer and view written to
- `VPC_GS_LAYER_CNTL`: same but for gs
- `VPC_DS_LAYER_CNTL`: same but for ds
- `VPC_VS_PACK`: where are pos and pointsize written to, total outs
- `VPC_GS_PACK`: same but for gs
- `VPC_DS_PACK`: same but for ds
- `VPC_CNTL_0`: where are primid and viewid written to, fs input counts
- `VPC_POLYGON_MODE`: polygon mode (fill/line/point)
- `VPC_VARYING_INTERP(i)`: how outs are interpolated for fs
- `VPC_VARYING_PS_REPL(i)`: for point sprites
- `VPC_POINT_COORD_INVERT`: invert point coord
- `VPC_VAR_DISABLE(i)`: which outs can be disabled
- `VPC_SO_CNTL`
- `VPC_SO_PROG`
- `VPC_SO_STREAM_COUNTS`: addrs to write counts to
- `VPC_SO_*(i)`: num of components
- `VPC_SO_STREAM_CNTL`: stream-to-buffer mapping
- `VPC_SO_DISABLE`: disable SO (e.g., was enabled for binning pass already)

## GRAS, graphics rasterizer?

- `GRAS_BIN_CONTROL`: bin w/h and flags
- `GRAS_VS_CL_CNTL`: clip masks
- `GRAS_VS_LAYER_CNTL`: writes layer/view 
- `GRAS_DS_CL_CNTL`: same but for ds
- `GRAS_DS_LAYER_CNTL`: same but for ds
- `GRAS_GS_CL_CNTL`: same but for gs
- `GRAS_GS_LAYER_CNTL`: same but for gs
- `GRAS_CL_CNTL`: depth clip
- `GRAS_CL_GUARDBAND_CLIP_ADJ`: viewport guardband
- `GRAS_CL_VPORT_*`: viewports
- `GRAS_CL_Z_CLAMP_*`: depth clamp
- `GRAS_SC_CNTL`: rasterization order
- `GRAS_SC_SCREEN_SCISSOR_*`: scissors
- `GRAS_SC_VIEWPORT_SCISSOR_*`: viewports
- `GRAS_SC_WINDOW_SCISSOR_*`: cur tile coords
- `GRAS_MAX_LAYER_INDEX`: max fb layer
- `GRAS_DBG_ECO_CNTL`: magic
- `GRAS_SU_CNTL`: cull, ccw, line width, smooth line, etc.
- `GRAS_SU_CONSERVATIVE_RAS_CNTL`: conservative rast
- `GRAS_SU_DEPTH_BUFFER_INFO`: depth buffer fmt
- `GRAS_SU_DEPTH_PLANE_CNTL`: early-z, lrz, or late-z
- `GRAS_SU_POINT_MINMAX`: point min/max
- `GRAS_SU_POINT_SIZE`: point size
- `GRAS_SU_POLY_OFFSET_OFFSET`: depth bias
- `GRAS_SU_POLY_OFFSET_OFFSET_CLAMP`: depth bias
- `GRAS_SU_POLY_OFFSET_SCALE`: depth bias
- `GRAS_LRZ_BUFFER_BASE`: lrz bo addr
- `GRAS_LRZ_BUFFER_PITCH`: lrz pitch
- `GRAS_LRZ_FAST_CLEAR_BUFFER_BASE`: lrz fast clear addr
- `GRAS_LRZ_CNTL`: enable, write enable, test enable, depth bounds
- `GRAS_LRZ_MRT_BUF_INFO_0`: MRT0 format
- `GRAS_LRZ_PS_INPUT_CNTL`: generates sampleid
- `GRAS_RAS_MSAA_CNTL`: msaa
- `GRAS_DEST_MSAA_CNTL`: msaa
- `GRAS_SAMPLE_CNTL`: per sample mode
- `GRAS_SAMPLE_CONFIG`: sample locs
- `GRAS_CNTL`: barycentric interps
- `GRAS_2D_BLIT_CNTL`: format, scissor, rotate, etc.
- `GRAS_2D_SRC_BR_X`: blit src coords
- `GRAS_2D_DST_*`: blit dst coords
- `GRAS_2D_RESOLVE_CNTL_1`: cur tile origin
- `GRAS_2D_RESOLVE_CNTL_2`: cur tile size

## RB, render buffer?

- `RB_BIN_CONTROL`: bin w/h and flags
- `RB_BIN_CONTROL2`: bin w/h
- `RB_CCU_CNTL`: set gmem offset for CCU cache
- `RB_FS_OUTPUT_CNTL0`: fs outputs pos, samplemask, stencilref
- `RB_FS_OUTPUT_CNTL1`: mrt count
- `RB_ALPHA_CONTROL`: alpha test
- `RB_STENCIL_*`: stencil format, pitch, layer, bo addr, gmem offset
- `RB_STENCIL_CONTROL`: stencil enable/funcs
- `RB_STENCILMASK`: stencil mask
- `RB_STENCILREF`: stencil ref
- `RB_STENCILWRMASK`: stencil writemask
- `RB_DEPTH_BUFFER_*`: format, pitch, layer, bo addr, gmem offset 
- `RB_DEPTH_CNTL`: depth test enable, compare op, depth write/clamp/bounds
- `RB_DEPTH_FLAG_BUFFER_BASE`: addr of ubwc metadata
- `RB_DEPTH_PLANE_CNTL`: early-z, lrz, or late-z
- `RB_LRZ_CNTL`: lrz enable
- `RB_Z_BOUNDS_MAX`: depth bounds
- `RB_Z_BOUNDS_MIN`: depth bounds
- `RB_Z_CLAMP_MAX`: depth clamp
- `RB_Z_CLAMP_MIN`: depth clamp
- `RB_SAMPLE_COUNT_ADDR`: where to write zpass count, for queries
- `RB_SAMPLE_COUNT_CONTROL`: how to write zpass count, for queries
- `RB_MRT_*`: mrt format, pitch, layer, bo addr, gmem offset
- `RB_MRT_FLAG_BUFFER_*`: addr of ubwc metadata
- `RB_RENDER_COMPONENTS`: bitmask of which compoents of which mrts are written
- `RB_RENDER_CONTROL0`: barycentric interps
- `RB_RENDER_CONTROL1`: samplemask, sampleid, faceness
- `RB_BLEND_CNTL`: blend enable mask, dual color, alpha-to-coverage, etc.
- `RB_BLEND_RED_F32`: blend const color
- `RB_BLEND_GREEN_F32`: blend const color
- `RB_BLEND_BLUE_F32`: blend const color
- `RB_BLEND_ALPHA_F32`: blend const color
- `RB_RAS_MSAA_CNTL`: msaa
- `RB_DEST_MSAA_CNTL`: msaa
- `RB_MSAA_CNTL`: msaa
- `RB_DITHER_CNTL`: enable dither
- `RB_RENDER_CNTL`: enable ubwc writes
- `RB_SAMPLE_CNTL`: per sample mode
- `RB_SAMPLE_CONFIG`: sample locs
- `RB_SRGB_CNTL`: which MRTs have sRGB
- `RB_WINDOW_OFFSET`: cur tile origin
- `RB_WINDOW_OFFSET2`: cur tile origin
- `RB_2D_BLIT_CNTL`: format, scissor, rotate, etc.
- `RB_2D_DST_*`: blit dst coords
- `RB_2D_DST_INFO`: dst format, bo addr, pitch
- `RB_2D_DST_FLAGS`: dst ubwc metadata addr
- `RB_2D_SRC_SOLID_C0`: clear value
- `RB_2D_UNKNOWN_8C01`: preserve depth or stencil values
- `RB_BLIT_BASE_GMEM`: dst gmem offset
- `RB_BLIT_CLEAR_COLOR_DW*`: clear value
- `RB_BLIT_DST_INFO`: dst format, bo addr, pitch
- `RB_BLIT_FLAG_DST`: dst ubwc metadata addr
- `RB_BLIT_INFO`: src or dst is gmem, clear mask
- `RB_BLIT_SCISSOR_*`: scissor

## SP, streaming processor

- `SP_VS_CONFIG`:
- `SP_VS_CTRL_REG0`:
- `SP_VS_INSTRLEN`:
- `SP_VS_OBJ_FIRST_EXEC_OFFSET`:
- `SP_VS_OUT_REG`:
- `SP_VS_PRIMITIVE_CNTL`:
- `SP_VS_PVT_MEM_HW_STACK_OFFSET`:
- `SP_VS_PVT_MEM_PARAM`:
- `SP_VS_PVT_MEM_SIZE`:
- `SP_VS_VPC_DST_REG`:
- `SP_HS_BRANCH_COND`:
- `SP_HS_CONFIG`:
- `SP_HS_CTRL_REG0`:
- `SP_HS_INSTRLEN`:
- `SP_HS_OBJ_FIRST_EXEC_OFFSET`:
- `SP_HS_PVT_MEM_HW_STACK_OFFSET`:
- `SP_HS_WAVE_INPUT_SIZE`:
- `SP_DS_CONFIG`:
- `SP_DS_CTRL_REG0`:
- `SP_DS_INSTRLEN`:
- `SP_DS_OBJ_FIRST_EXEC_OFFSET`:
- `SP_DS_OUT_REG`:
- `SP_DS_PRIMITIVE_CNTL`:
- `SP_DS_PVT_MEM_HW_STACK_OFFSET`:
- `SP_DS_VPC_DST_REG`:
- `SP_GS_CONFIG`:
- `SP_GS_CTRL_REG0`:
- `SP_GS_INSTRLEN`:
- `SP_GS_OBJ_FIRST_EXEC_OFFSET`:
- `SP_GS_OUT_REG`:
- `SP_GS_PRIMITIVE_CNTL`:
- `SP_GS_PRIM_SIZE`:
- `SP_GS_PVT_MEM_HW_STACK_OFFSET`:
- `SP_GS_VPC_DST_REG`:
- `SP_FS_BINDLESS_PREFETCH_CMD`:
- `SP_FS_CONFIG`:
- `SP_FS_CTRL_REG0`:
- `SP_FS_INSTRLEN`:
- `SP_FS_MRT_REG`:
- `SP_FS_OBJ_FIRST_EXEC_OFFSET`:
- `SP_FS_OUTPUT_CNTL0`:
- `SP_FS_OUTPUT_CNTL1`:
- `SP_FS_OUTPUT_REG`:
- `SP_FS_PREFETCH_CMD`:
- `SP_FS_PREFETCH_CNTL`:
- `SP_FS_PVT_MEM_HW_STACK_OFFSET`:
- `SP_FS_RENDER_COMPONENTS`:
- `SP_FS_TEX_CONST`:
- `SP_FS_TEX_COUNT`:
- `SP_FS_TEX_SAMP`:
- `SP_CS_CNTL_0`:
- `SP_CS_CNTL_1`:
- `SP_CS_CONFIG`:
- `SP_CS_CTRL_REG0`:
- `SP_CS_INSTRLEN`:
- `SP_CS_OBJ_FIRST_EXEC_OFFSET`:
- `SP_CS_PVT_MEM_HW_STACK_OFFSET`:
- `SP_CS_UNKNOWN_A9B1`: shared size
- `SP_2D_DST_FORMAT`: 2d blit dst format
- `SP_BLEND_CNTL`: blend enable, dual color, alpha-to-coverage
- `SP_CHICKEN_BITS`: magic
- `SP_FLOAT_CNTL`: alt float mode
- `SP_IBO_COUNT`: 0
- `SP_MODE_CONTROL`: how float consts are converted to half
- `SP_PERFCTR_ENABLE`: magic
- `SP_PS_2D_SRC`:
- `SP_PS_2D_SRC_FLAGS`:
- `SP_PS_2D_SRC_INFO`:
- `SP_PS_2D_SRC_PITCH`:
- `SP_PS_2D_SRC_SIZE`:
- `SP_PS_TP_BORDER_COLOR_BASE_ADDR`:
- `SP_SRGB_CNTL`: which MRTs are srgb
- `SP_TP_BORDER_COLOR_BASE_ADDR`: border color
- `SP_TP_MODE_CNTL`: gl/d3d mode
- `SP_TP_RAS_MSAA_CNTL`: msaa
- `SP_TP_DEST_MSAA_CNTL`: msaa
- `SP_TP_SAMPLE_CONFIG`: sample locations
- `SP_TP_WINDOW_OFFSET`: cur tile origin
- `SP_WINDOW_OFFSET`: cur tile origin

## HLSQ

- `HLSQ_VS_CNTL`:
- `HLSQ_HS_CNTL`:
- `HLSQ_DS_CNTL`:
- `HLSQ_GS_CNTL`:
- `HLSQ_FS_CNTL`:
- `HLSQ_FS_CNTL_0`:
- `HLSQ_CS_CNTL`:
- `HLSQ_CS_CNTL_0`:
- `HLSQ_CS_CNTL_1`:
- `HLSQ_CS_KERNEL_GROUP_X`:
- `HLSQ_CS_KERNEL_GROUP_Y`:
- `HLSQ_CS_KERNEL_GROUP_Z`:
- `HLSQ_CS_NDRANGE_0`:
- `HLSQ_CS_NDRANGE_1`:
- `HLSQ_CS_NDRANGE_2`:
- `HLSQ_CS_NDRANGE_3`:
- `HLSQ_CS_NDRANGE_4`:
- `HLSQ_CS_NDRANGE_5`:
- `HLSQ_CS_NDRANGE_6`:
- `HLSQ_CS_UNKNOWN_B9D0`:
- `HLSQ_CONTROL_1_REG`:
- `HLSQ_CONTROL_2_REG`:
- `HLSQ_CONTROL_3_REG`:
- `HLSQ_CONTROL_4_REG`:
- `HLSQ_CONTROL_5_REG`:
- `HLSQ_INVALIDATE_CMD`:
- `HLSQ_SHARED_CONSTS`:

## Misc Blocks

- RBBM: ring buffer status?
- DBGC: gpu status for debug?
- UCHE: unified cache / L2
- TPL1: texture process / L1