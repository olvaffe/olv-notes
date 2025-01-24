Qualcomm Adreno
===============

## History

- <https://en.wikipedia.org/wiki/Adreno>
- `freedreno_devices.py`
- A2XX
  - Adreno 200
  - Adreno 201
  - Adreno 205
  - Adreno 220
- A3XX
  - Adreno 305
  - Adreno 305B
  - Adreno 306A
  - Adreno 307
  - Adreno 320
  - Adreno 330
- A4XX
  - Adreno 405
  - Adreno 420
  - Adreno 430
- A5XX
  - Adreno 505
  - Adreno 506
  - Adreno 508
  - Adreno 509
  - Adreno 510
  - Adreno 512
  - Adreno 530
  - Adreno 540
- A6XX Gen1
  - Adreno 605
  - Adreno 608
  - Adreno 610
  - Adreno 612
  - Adreno 615
  - Adreno 616
  - Adreno 618
  - Adreno 619
  - Adreno 620
  - Adreno 630
- A6XX Gen2
  - Adreno 640
  - Adreno 680
- A6XX Gen3
  - Adreno 621
  - Adreno 650
- A6XX Gen4
  - Adreno 635
  - Adreno 643
  - Adreno 660
  - Adreno 662
  - Adreno 663
  - Adreno 690
- A7XX Gen1
  - Adreno 725
  - Adreno 730
- A7XX Gen2
  - Adreno 735
  - Adreno 740
- A7XX Gen3
  - Adreno 750

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

- CP used to consist of PFP (prefetch parser) and ME (microengine)
  - PFP fetches commands into a FIFO named MEQ
  - ME processes commands from MEQ
  - some commands are executed by PFP and are not fetched into MEQ
  - they are different microcontrollers with different instruction sets before
    a5xx
  - in a5xx, they are two of the same microcontrollers using an instruction
    set nicknamed afuc
- since a6xx, CP is a microcontroller called SQE
  - SQE uses afuc instruction set
  - SQE has no direct memory access.  It relies on GPU core or ROQ for memory
    access
  - when idle, SQE executes `waitin` waiting for packets to arrive at ROQ
  - ROQ peaks the packets and put them onto different ROQ queues
    - RB
    - IB1 
    - IB2
    - MRB (memory read)
    - SDS (set draw state)
    - VSD (visibility stream decode)
  - SQE reads packets from ROQ and executes them
    - SQE is capable of ALU, synchronized gpu register access, and pipelined
      gpu register access
    - it has no memory access; it uses the gpu (by programming gpu registers)
      to access memory
  - see also
    - `afuc/README.rst`
    - `registers/adreno/adreno_pm4.xml`
    - <https://www.x.org/docs/AMD/>
    - `dump_cmdstream` in crashdec
- on gpu fault, kernel prints
  `gpu fault ring 0 fence 36b0 status 00800005 rb 04f0/0560 ib1 0000000113403000/0002 ib2 0000000113413000/0000`
  - `0x36b0` is the fence seqno of the last job
    - each job is assigned a fence seqno
  - `0x00800005` is `REG_A6XX_RBBM_STATUS`
    - `GPU_BUSY_IGN_AHB`
    - `CP_BUSY`
    - `CP_AHB_BUSY_CX_MASTER`
  - `0x04f0` is `REG_A6XX_CP_RB_RPTR`
    - this is the read ptr of the ring buffer
  - `0x0560` is `REG_A6XX_CP_RB_WPTR`
    - this is the write ptr of the ring buffer
  - `0x0000000113403000` is `REG_A6XX_CP_IB1_BASE`
    - this is the iova of an execbuf
    - `CP_INDIRECT_BUFFER_PFE(iova, size)`
  - `0x0002` is `REG_A6XX_CP_IB1_REM_SIZE`
    - this is the remaining size in IB1
  - `0x0000000113413000` is `REG_A6XX_CP_IB2_BASE`
    - execbuf can have another level of indirection known as IB2
      - it is used by userspace driver
      - e.g., draw ib that is invoked for each tile
    - `CP_INDIRECT_BUFFER(iova, size)`
  - `0x0000` is `REG_A6XX_CP_IB2_REM_SIZE`
    - this is the remaining size in IB2
  - `REG_A6XX_CP_CSQ_IB1_STAT` and `REG_A6XX_CP_CSQ_IB2_STAT`
    - ROQ pre-fetches commands and updates `REG_A6XX_CP_IB1_REM_SIZE`
    - `REG_A6XX_CP_CSQ_IB1_STAT` is the size of data prefetched and yet
      consumed by SQE
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
  - `CP_WAIT_FOR_IDLE` tells ME to stall `CP_DRAW_*` until the pipeline is
    idle; other commands can go on
  - `CP_WAIT_MEM_WRITES` tells ME to stall `CP_DRAW_*` until mem writes from
    ME have completed; other commands can go on
- memory access
  - CP memory access is uncached
  - `CP_MEM_WRITE` writes a dword or qword to addr
  - `CP_MEM_TO_MEM` sums  dwords or qwords and writes the result to addr
    - `DOUBLE` selects dword / qword
    - `NEG_A/B/C` negates values before summing
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
      - e.g., binning must decide no draw hit a tile and stateobjs can be
      	skipped
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

## CP Firmware

- `a630_sqe.fw` can be disassembled with `afuc-disasm`
  - `disasm` executes and disassembles all instructions
    - `./afuc-disasm /lib/firmware/qcom/a630_sqe.fw`
  - lpac is not used
  - `emu_pipe_regs` is not used
- afuc emulator
  - 12-bit Control Register Space
    - `struct emu_control_regs`
    - `emu_set_control_reg`
    - `emu_get_control_reg`
    - they emulate SQE internal registers and only a few are used/known
      - accessed using `cread` and `cwrite` instructions
      - 0x0xx: 
      - 0x1xx: scratch space
  - 8-bit Pipe Register Space
    - `struct emu_pipe_regs`
    - `emu_set_pipe_reg`
    - `emu_get_pipe_reg`
    - they emulate pipe registers
      - write addr first to `$addr` and write val to `$data`
        - addr is left-shifted by 24
        - addr is auto-incremented after write
        - some pipe regs are "void" and does not need writing val to `$data`
      - write-only
  - 16-bit GPU Register Space
    - `struct emu_gpu_regs`
    - `emu_set_gpu_reg`
    - `emu_get_gpu_reg`
    - they emulate gpu registers
      - write addr first to `$addr` and write val to `$data`
        - addr is auto-incremented after write
      - or, cwrite addr first to `REG_WRITE_ADDR` and cwrite val to
        `REG_WRITE`
        - addr is auto-incremented after write
      - to read, cwrite count first to `REG_READ_DWORDS` and cwrite addr to
        `REG_READ_ADDR`
        - this reads the vals into a FIFO which can be accessed via `$regdata`
  - 32 GPRs
    - `struct emu_gpr_regs`
    - `emu_set_gpr_reg`
    - `emu_get_gpr_reg`
    - some of them are special and are called fifo regs
      - `emu_set_fifo_reg`
      - `emu_get_fifo_reg`
- `emu_run_bootstrap` executes the firmware up the point that the packet table
  is initialized
  - read gpu registers
    - `cwrite`s to SQE's `REG_READ_DWORDS` and `REG_READ_ADDR`
      - this reads the gpu registers into a fifo
    - reads from `REG_REGDATA` gpr
      - each read consumes a value in the fifo
  - write gpu registers
    - `cwrite`s to SQE's `REG_WRITE_ADDR` and `REG_WRITE`
      - this writes the gpu registers
      - each write increments `REG_WRITE_ADDR`
  - write gpu registers #2
    - writes to `REG_ADDR` or `REG_USRADDR`
    - writes to `REG_DATA`
      - this gets the gpu reg from `REG_ADDR` or `REG_USRADDR` and increments
      	them
      - this then updates the gpu reg
  - read cpu memory
    - `cwrite`s to SQE's `MEM_READ_ADDR` and `MEM_READ_DWORDS`
      - this reads the cpu memory data into a fifo
      - cpu mem addr is 64-bit iova and requires two `cwrite`s
    - reads from `REG_MEMDATA` gpr
      - each read consumes a value in the fifo
  - read cpu memory #2
    - `cwrite`s to SQE's `LOAD_STORE_HI`
      - this writes the higher 32-bit of iova
    - `load`s from the iova
      - this directly specifies the lower 32-bit of iova plus an offset
  - read cpu memory #3
    - write `NRT_ADDR` pipe registers and read from `REG_MEMDATA`
  - write cpu memory
    - for small writes, write `NRT_ADDR` and `NRT_DATA` pipe registers
  - initialize packet table
    - `cwrite`s 0 to SQE's `PACKET_TABLE_WRITE_ADDR`
    - `cwrite`s to `PACKET_TABLE_WRITE` with `(rep)`
      - `(rep)` repeats the instruction until `REG_REM` is 0
      - each write increments the addr
      - each write also initializes `emu->jmptbl[addr]`
- with the packet table known, `afuc-disasm` disassembles all instructions and
  labels where each packet type is handled
- if emulator mode, `afuc-disasm` keeps executing while disassembling
  - `waitin` waits for the next PM4 packet
    - emulator sets `emu->waitin` and clears `emu->run_mode`
  - `emu_main_prompt` waits for user input
    - user should inject a PM4 packet into `emu->roq`, which is the command
      queue
  - `mov $01, $data` always follows `waitin`
    - each read from `REG_DATA` consumes `emu->roq`
    - `emu_step` also has logic to
      - peek the header in `$01`
      - update `REG_REM`
      - jumps with `emu->gpr_regs.pc = emu->jmptbl[id]`
- for example, `CP_CONTEXT_REG_BUNCH` writes to a bunch of gpu registers
  - `(rep)(xmov3)mov $usraddr, $data`
  - reading `REG_DATA` reads the next dword in ROQ
  - writing `REG_USRDATA` updates the reg addr
  - `(xmov3)` does `min(3, REG_REM)` more `mov`s from src2 to `$data`
  - `(rep)` repeats until `REG_MEM` is 0.  That is, all `$data` are consumed
- `CP_WAIT_FOR_ME`
  - increment `WFI_PEND_CTR` with `cwrite` to `WFI_PEND_INCR` control reg
    - I think this happens immediately
  - decrement `WFI_PEND_CTR` with `mov` to `WFI_PEND_DECR` pipe reg
    - I think this is pipelined
  - loop until `WFI_PEND_CTR` is 0
- `CP_WAIT_FOR_IDLE`
  - increment `WFI_PEND_CTR` immediately
  - decrement `WFI_PEND_CTR` pipelined
- `CP_REG_TO_MEM`
  - it calls `fxn08`
    - increment `WFI_PEND_CTR` immediately
    - decrement `WFI_PEND_CTR` pipelined
    - calls `fxn06` to loop until `WFI_PEND_CTR` is 0
- `CP_MEM_WRITE`
  - `mov` to `NRT_ADDR` and `NRT_DATA` pipe registers, which is pipelined
- `CP_WAIT_MEM_WRITES`
  - `mov` to `WAIT_MEM_WRITES` pipe reg, which is pipelined
- `CP_BLIT`
  - writes `CP_2D_EVENT_START`
  - writes `HLSQ_2D_EVENT_CMD` and `PC_2D_EVENT_CMD`
    - `BLIT_OP_FILL_2D`
    - `BLIT_OP_COPY_2D`
    - `BLIT_OP_SCALE_2D`
  - writes coordinates
  - writes to `HLSQ_2D_EVENT_CMD` and `PC_2D_EVENT_CMD` again
    - `CONTEXT_DONE_2D`
  - writes `CP_2D_EVENT_END`
    - this triggers the event
- `CP_SKIP_IB2_ENABLE_LOCAL`
  - `$14` bit 8 indicates local ib2 skip
- `CP_SKIP_IB2_ENABLE_GLOBAL`
  - `$14` bit 9 indicates global ib2 skip
- `CP_SET_MODE`
  - if 1, set b29 and b30 of `$14` and done
  - if 0, clear b29 and optionally b30 of `$14`
  - `$12`
    - b0 and b1 are the IB level: 0, 1, 2, or 3
      - 0 is ring buffer
      - 1 is user batches
      - 2 is secondary batches or stateobjs
      - 3 is stateobjs in secondary batches
    - b4 and b5 are flags
- `CP_SET_MARKER`
  - userspace only uses mode 1..8
- `CP_INDIRECT_BUFFER`
- `CP_PREEMPT_ENABLE`
  - infinite loop
  - it seems the firmware jumps here for fatal errors
- kernel
  - `a6xx_hw_init`
    - writes `CP_SQE_CNTL` to start SQE
    - constructs a `CP_ME_INIT` packet and writes `CP_RB_WPTR` to exec
    - more init
  - `a6xx_submit`
    - `CP_EVENT_WRITE(PC_CCU_INVALIDATE_DEPTH)`
    - `CP_EVENT_WRITE(PC_CCU_INVALIDATE_COLOR)`
    - one or more `CP_INDIRECT_BUFFER_PFE`
    - write fence seqno to `CP_SCRATCH_REG(2)` (for hang debug)
    - `CP_EVENT_WRITE(CACHE_FLUSH_TS)` w/ interrupt and writing seqno to
      `msm_rbmemptrs::fence` 
- more `a630_sqe.fw`
  - `fxn08` is essentially `CP_WAIT_FOR_ME`
  - `fxn09` writes some bits of `$13` to control reg `0x0a01`
  - `fxn10`
    - cwrites `$00` (i.e., 0) to control reg `0x05b`
    - moves `$08` (i.e., where reg addr is stored) `$usraddr` with bit 20 set
    - moves `$09` (i.e., where count is stored) `$data`
    - keeps creading `0x05b` to `$02` until bit 0 is set
    - cwrites `$00` to control reg `0x05b` again
    - I think this checks if all regs in `$08` to `$09` are accessible
      - bit 0 in `$02` means done
      - bit 2 in `$02` means error
      - if bit 2 is not set, `$02` is cleared
  - `l098`
    - cwrites `CP_PROTECT_CNTL` to `REG_READ_ADDR`
    - cwrites `0x10` to `0x064`
    - jumps to `CP_NOP`
  - `CP_REG_TO_SCRATCH`
    - calls `fxn08` for WFM
    - calls `fxn10` to validate regs
    - jumps to `l098` if validation fails
    - cwrites to `REG_READ_DWORDS` and `REG_READ_ADDR`
    - cwrites `$regdata` to `@SCRATCH_REG0` count times
  - `CP_SCRATCH_TO_REG`
    - `$08` is reg addr
    - `$usraddr` is reg addr with bit 18 potentially set
    - `$02` is count
    - `$03` is scratch reg offset
    - creads `@SCRATCH_REG0` to `$data`
  - `$12` appears to shadow `IB_LEVEL`
  - `$13`
    - bit 0-3: conditional rendering
      - `CP_DRAW_PRED_ENABLE_GLOBAL` sets it to `b1110` or 0
      - `CP_DRAW_PRED_ENABLE_LOCAL` sets/clears bit 3
      - `CP_DRAW_PRED_SET` sets/clears bit 0 and 1
    - bit 6
    - bit 7
    - bit 8: skip ib2 local
    - bit 9: skip ib2 global
    - bit 10: predicate
      - `CP_REG_TEST` sets/clears the bit (true/false)
      - `CP_COND_REG_EXEC` tests the bit

## Caches

- L1 ro texture cache, coupled with TP
  - event `CACHE_INVALIDATE` invalidates both UCHE and L1
  - there is no control over L1 otherwise
- CCU rw color/depth caches, coupled with RB
  - some socs have multiple CCUs
  - CCUs use GMEM as its storage
    - each CCU color cache needs 16KB, `A6XX_CCU_GMEM_COLOR_SIZE`
    - each CCU depth cache needs 64KB, `A6XX_CCU_DEPTH_SIZE`
  - `RB_CCU_CNTL`
    - `GMEM` is set when gmem rendering; otherwise, sysmem rendering
      - it might sound pointless to use GMEM as caches when doing gmem
        rendering.  But CCUs use the color caches for a different purpose.
    - `DEPTH_OFFSET` is set to 0, and is not used when gmem rendering
    - `COLOR_OFFSET` is always used
  - for sysmem rendering,
    - event `PC_CCU_FLUSH_COLOR_TS` flushes CCU color caches
    - event `PC_CCU_FLUSH_DEPTH_TS` flushes CCU depth caches
    - event `PC_CCU_INVALIDATE_COLOR` invalidates CCU color caches
    - event `PC_CCU_INVALIDATE_DEPTH` invalidates CCU depth caches
  - for gmem rendering
    - event `PC_CCU_RESOLVE_TS` flushes CCU color caches (after tile stores)
- UCHE rw cache
  - UCHE is L2 and is used by the entire GPU pipeline
    - it is skipped by CP
    - CCU likely skips UCHE
      - turnip says so
  - event `CACHE_FLUSH_TS` flushes UCHE
  - event `CACHE_INVALIDATE` invalidates both UCHE and L1
- in other words,
  - shader core texturing uses L1
  - RB uses CCU caches
    - there are several ways to use RB though
    - in sysmem rendering, CCU caches are color/depth caches
    - in gmem rendering, CCU caches are resolve (tile store) caches
  - `CP_BLIT` uses a part of gpu pipeline, from GRAS to RB
    - it textures from src and uses L1 cache
    - it sysmem renders to dst and uses CCU caches
  - event `BLIT` uses a part of gpu pipeline, only RB to be exact
    - tile load reads from src and likely is uncached
      - turnip says so, instead of uisng UCHE (or TP)
    - tile store writes to dst and uses CCU color caches (for other purposes)
  - all other gpu memory access uses UCHE
  - L1 also use UCHE because UCHE is L2
  - CCU likely skips UCHE
    - turnip says so
  - CP memory access is uncached
- all the FLUSH or INVALIDATE events travel down the gpu pipeline and takes
  place at the end of the gpu pipeline
  - this avoids using `CP_WAIT_FOR_IDLE`
- `GRAS_SC_CNTL` has a field for overlapping primitives
  - `NO_FLUSH` takes no action for overlapping primitives
  - `FLUSH_PER_OVERLAP` stalls and invalidates UCHE for overlapping primitives
    - this is needed when there is a feedback loop
    - sample from a MRT for advanced blending, etc.
  - `FLUSH_PER_OVERLAP_AND_OVERWRITE` addtionally keeps CCU and UCHE in sync
    - this is needed when there is a feedback loop and we are doing sysmem
      rendering

## Pipeline Clusters

- the pipeline is splitted into clusters (stages) running independently
  - `FE`
  - `SP_VS`
  - `PC_VS`
  - `GRAS`
  - `SP_FS`
  - `PS`
- draws and computes are pipelined
  - earlier stages can run draw N+1 while later stages still run draw N
  - WFI is a full pipeline stall, used when earlier stages need the results of
    later stages
- each cluster has a pair of register banks
  - CP can update bank A while the cluster uses bank B
  - not all registers are banked
- CP can access gpu registers immediately or pipelined
  - it can read/write gpu registers immediately
  - it can write gpu registers pipelined
    - `CP_MEMPOOL` has 6 queues for each clusters
    - pipelined writes are pushed into the corresponding queues
    - clusters will fetch the writes from the queues
- CP uses pipelined register writes to control clusters
  - a write to `CP_EVENT_START` clears the done bit of the current context
    (bank)
  - a write to `{PC,HLSQ}_DRAW_CMD` kicks start a draw
  - a write to `{PC,HLSQ}_EVENT` kicks start a event
    - `vgt_event_type` is the types of events
    - a `CONTEXT_DONE` event waits for the previous work and sets the done bit
  - a write to `CP_EVENT_END` waits on the done bit for the other context,
    copies dirtied registers over, and switches to the other context
- IOW, a cluter sees this sequence of commands from its queue
  - context A: state updates
  - context A: `CP_EVENT_START`
  - context A: draw
  - context A: `CP_EVENT_END` to wait for context B (whose registers might still be used
               for drawing) and switch over
  - context B: state updates
  - context B: `CP_EVENT_START`
  - context B: draw
  - context B: `CP_EVENT_END` to wait for context A and switch
- `CP_EXEC_CS`
  - writes `0x3` to `EVENT_CMD` pipe reg
    - `0x3` means `DISPATCH`
  - writes the lower 8 bits of `$13` to `CP_EVENT_START`
    - normally 0
  - calls `fxn01`
    - reads `MARKER` immediately
    - clears some bits
    - writes `MARKER` immediately
    - writes `PC_MARKER` pipelined
  - writes the lower 8 bits of `$13` to `HLSQ_DISPATCH_CMD`
  - writes the lower 8 bits of `$13` to `PC_DISPATCH_CMD`
  - calls `fxn02`
    - writes `CONTEXT_DONE` to `HLSQ_EVENT_CMD`
    - writes `CONTEXT_DONE` to `PC_EVENT_CMD`
    - writes to `PC_MARKER`
    - writes to `CP_EVENT_END`

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
  - `RB_DEPTH_BUFFER_INFO` controls format
  - `RB_DEPTH_BUFFER_PITCH` controls pitch
  - `RB_DEPTH_BUFFER_ARRAY_PITCH` controls layer pitch
  - `RB_DEPTH_BUFFER_BASE` controls iova
  - `RB_DEPTH_BUFFER_BASE_GMEM` controls gmem offset
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
  - there are 8 MRTs and each MRT has 8 registers
  - `RB_MRT_CONTROL` controls blending, write mask, and logicop
  - `RB_MRT_BLEND_CONTROL` controls blend factors
  - `RB_MRT_BUF_INFO` controls the buffer format, tiling, swap
  - `RB_MRT_PITCH` controls the buffer pitch
  - `RB_MRT_ARRAY_PITCH` controls the layer pitch
  - 3 more registers control the iova and the gmem offset
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

- <https://gitlab.freedesktop.org/freedreno/freedreno/-/wikis/A6xx-SP>
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
- `SP_BINDLESS_BASE`: 5 bindless bases
- `SP_IBO_COUNT`: 0
- `SP_MODE_CONTROL`: how float consts are converted to half
- `SP_PERFCTR_ENABLE`: magic
- `SP_PS_2D_SRC_INFO`: src format, bo addr, pitch
- `SP_PS_2D_SRC_FLAGS`: src ubwc
- `SP_PS_TP_BORDER_COLOR_BASE_ADDR`: border color
- `SP_SRGB_CNTL`: which MRTs are srgb
- `SP_TP_BORDER_COLOR_BASE_ADDR`: border color
- `SP_TP_MODE_CNTL`: gl/d3d mode
- `SP_TP_RAS_MSAA_CNTL`: msaa
- `SP_TP_DEST_MSAA_CNTL`: msaa
- `SP_TP_SAMPLE_CONFIG`: sample locations
- `SP_TP_WINDOW_OFFSET`: cur tile origin
- `SP_WINDOW_OFFSET`: cur tile origin

## HLSQ

- `HLSQ_VS_CNTL`: enabled and const len
- `HLSQ_HS_CNTL`: same but for hs
- `HLSQ_DS_CNTL`: same but for ds
- `HLSQ_GS_CNTL`: same but for gs
- `HLSQ_FS_CNTL`: same but for fs
- `HLSQ_FS_CNTL_0`: threadsize, fs has inputs
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
- `HLSQ_CONTROL_[1-5]_REG`: regids of special fs inputs (face, sampleid, bary coeff)
- `HLSQ_BINDLESS_BASE`: 5 bindless bases
- `HLSQ_INVALIDATE_CMD`: invalidate vs/hs/ds/gs/fs/cs states
- `HLSQ_SHARED_CONSTS`: always 0

## 2D Clear and Blit

- `CP_BLIT` with `BLIT_OP_SCALE`
  - support memory clear or memory-to-memory blit
  - support rotation and scaling
  - support format conversion, tiles, and UBWC
    - there seems to be troubles with Y8 or S8 aspects of multi-aspect formats
    - format conversion is limited to compatible formats?
      - i doubt this
    - tiling and UBWC can be format-dependent.  We can't always reinterpret
      between compatible formats
  - ALWAYS resolve samples 
    - I guess its shader does not use txf
    - cannot be used when want to copy samples
- it uses these GRAS states
  - `GRAS_2D_BLIT_CNTL`
  - `GRAS_2D_DST_TL`
  - `GRAS_2D_DST_BR`
  - when `GRAS_2D_BLIT_CNTL::SOLID_COLOR` is not set,
    - `GRAS_2D_SRC_TL_X`
    - `GRAS_2D_SRC_TL_Y`
    - `GRAS_2D_SRC_BR_X`
    - `GRAS_2D_SRC_BR_Y`
  - GRAS is the rasterizer.  I guess it uses these 2D states to generate
    fragments and texcords
- it uses these SP states
  - `SP_2D_DST_FORMAT`
  - when `GRAS_2D_BLIT_CNTL::SOLID_COLOR` is not set
    - `SP_PS_2D_SRC_INFO`
    - `SP_PS_2D_SRC_SIZE`
    - `SP_PS_2D_SRC`
    - `SP_PS_2D_SRC_PITCH`
    - `SP_PS_2D_SRC_FLAGS`, if ubwc
    - `SP_PS_2D_SRC_FLAGS_PITCH`
    - I guess it uses a "PS" stage and executes texturing on the shader cores
- it uses these RB states
  - `RB_2D_BLIT_CNTL`, same as `GRAS_2D_BLIT_CNTL`
  - `RB_2D_SRC_SOLID_C0`, when `RB_2D_BLIT_CNTL::SOLID_COLOR` is set
  - `RB_2D_DST_INFO`
  - `RB_2D_DST`
  - `RB_2D_DST_PITCH`
  - `RB_2D_DST_FLAGS`, if ubwc
  - `RB_2D_DST_FLAGS_PITCH`

## GMEM Clear and Blit

- `CP_EVENT_WRITE` with `BLIT`
  - support gmem clear
  - support mem-to-gmem and gmem-to-mem blit
- it uses these RB states
  - `RB_UNKNOWN_88D0`
  - `RB_BLIT_SCISSOR_TL`
  - `RB_BLIT_SCISSOR_BR`
  - `RB_BIN_CONTROL2`, I guess
  - `RB_WINDOW_OFFSET2`, I guess
  - `RB_MSAA_CNTL`
  - `RB_BLIT_BASE_GMEM`
  - `RB_BLIT_DST_INFO`
  - `RB_BLIT_DST`
  - `RB_BLIT_DST_PITCH`
  - `RB_BLIT_FLAG_DST`
  - `RB_BLIT_FLAG_DST_PITCH`
  - `RB_BLIT_CLEAR_COLOR_DW0`
  - `RB_BLIT_INFO`, control blit direction

## 3D Viewports and Scissors

- relevant GRAS states
  - `GRAS_CL_VPORT_*` are app viewports
    - hw follows vk rules
    - `XOFFSET` and `YOFFSET` are viewport center
    - `XSCALE` and `YSCALE` are half width/height
    - `ZOFFSET` is min depth
    - `ZSCALE` is full depth range
  - `GRAS_CL_GUARDBAND_CLIP_ADJ_*` are guardband
    - Clipping in 3d is expensive.  We would rather let the primitives travel
      down the pipeline a bit more and let the rasterizer clip in 2d.
      Guardband decides if a primitive can travel down and bypasses 3d
      clipping.
    - see intel prm
  - `GRAS_CL_Z_CLAMP_*` clamps transformed z to the range if enabled
  - `GRAS_SC_VIEWPORT_SCISSOR_*` are set to the same values of viewports
    - used by the rasterizer to clip in 2d?
  - `GRAS_SC_SCREEN_SCISSOR_*` are app scissors
- relevant RB states
  - `RB_Z_CLAMP_*` clamps z to the range before... depth write?
- interaction with tiles
  - `GRAS_BIN_CONTROL` is set to tile w/h and flags, before tile rendering
  - `GRAS_SC_WINDOW_SCISSOR_*` are set to tile coords for each tile
  - `RB_BIN_CONTROL` is set to the same as `GRAS_BIN_CONTROL`
  - `RB_WINDOW_OFFSET` is set to tile origins for each tile
  - `SP_WINDOW_OFFSET` is set to the same as `RB_WINDOW_OFFSET`
  - `SP_TP_WINDOW_OFFSET` is set to the same as `RB_WINDOW_OFFSET`
- these are set, but I think they are irrelevant
  - `GRAS_2D_RESOLVE_CNTL_1` is for `CP_BLIT`
  - `GRAS_2D_RESOLVE_CNTL_2` is for `CP_BLIT`
  - `RB_BIN_CONTROL2` is for `BLIT` event
  - `RB_WINDOW_OFFSET2` is for `BLIT` event
  - `RB_BLIT_SCISSOR_*` is for `BLIT` event

## `CP_LOAD_STATE6_*`

- `STATE_BLOCK`
  - `SB6_xS_TEX` is for
    - sampler descriptors
    - texture descriptors
    - I guess these are accessed by TP (texture processor)
  - `SB6_xS_SHADER`
    - UBO descriptors
    - shader binaries
    - shader immediates
    - UBO data
    - push constants
    - driver-generated params
    - if CS, SSBO/IBO descriptors
    - I guess these are accessed by SP (streaming processor)
  - `SB6_IBO` is for SSBO/IBO descriptors
    - I guess these are accessed by SP too, and they are shared by all
      graphics stages
  - `SB6_CS_IBO` not currently used; `SB6_CS_SHADER` is used instead
- `STATE_TYPE`
  - `ST6_SHADER`: the data is
    - SSBO/IBO descriptors for graphics stages
    - sampler descriptors
    - shader binaries
  - `ST6_CONSTANTS`: the data is
    - texture descriptors
    - shader immediates
    - UBO data
    - push constants
    - driver-generated params
  - `ST6_UBO`: the data is UBO descriptors
  - `ST6_IBO`: the data is SSBO/IBO descriptors for CS stage
- `STATE_SRC`
  - `SS6_DIRECT`: the payload contains the data to be loaded
  - `SS6_INDIRECT`: the payload has iova of the data; load does not happen
    until `CP_DRAW` and can be canceled by `HLSQ_INVALIDATE_CMD`
  - `SS6_BINDLESS`: the payload has bindless register id and an offset; load
    does not happen until `CP_DRAW` and can be canceled
  - `SS6_UBO`: not currently used
- bindless
  - there are 5 registers (`SP_BINDLESS_BASE` and `HLSQ_BINDLESS_BASE`) that
    can point to the base addresses of 5 descriptor sets.
  - the shader find a descriptor in a set by calculating base plus offset
  - the descriptors can be preloaded too

## `CP_SET_DRAW_STATE`

- there can be up to 32 driver-defined draw state groups
- each mode, `GMEM`, `SYSMEM`, or `BINNING`, has its own 32 draw state groups
- the draw state groups are executed on `CP_DRAW_*` or `CP_SET_MODE`
- the hw skips executing a draw state group when it is clean
  - that is, the draw state group has been executed and the iova hasn't
    changed
  - the `DIRTY` bit can force a draw state group dirty

## Formats

- Tiling
  - `TILE6_LINEAR` is linear
  - `TILE6_2` is what gmem uses
  - `TILE6_3` is tiled
  - for a tiled mipmap, it is controllable whether the tail is also tiled or
    linear
    - by default, the sampler hw assumes lod of width < 16 is linear unless
      `TILE_ALL` is set
  - MRT
    - tiling and ubwc are controllable
      - ubwc requires tiling
    - if the width is less than 16, it is preferred to use `TILE6_LINEAR`
      - this is assumed by the sampler hw unless `TILE_ALL` is set
      - because ubwc requires tiling, we use tiling even when the width is less
        than 16 when ubwc is enabled
  - Depth
    - always tiled
      - must set `TILE_ALL` when sampling a depth texture
    - ubwc is controllable
- UBWC
  - it is called flags in register definitions
- when linear and no ubwc, compatible formats can be reinterpreted
- otherwise, tiling and ubwc can change their behaviors depending on formats
- in gmem, images are tiled and non-ubwc
- in memory, images can be linear+non-ubwc, tiled+non-ubwc, or tiled+ubwc
  - but freedreno does not use tiled+non-ubwc
- `TILE6_3`
  - a macrotile is always 256 bytes
    - at cpp 16, it covers 4x4 pixels where each 2x2 pixels are packed
      together
      - in z-order space filling curve
    - at cpp 8, it covers 8x4 pixels in z-order
    - at cpp 4, it covers 16x4 pixels in z-order
  - images are covered by macrotiles in a very unique order
    - it is a strange-looking space filling curve
- UBWC
  - there is 1 byte for each 256-byte macrotile
    - there are also padding bytes for alignment
  - if bit 4 is set, the macrotile is not fast-cleared
    - the lower 4 bits plus 1 is the compressed size of the macrotile in 16
      bytes units
    - i.e., 0x11 means 32 bytes
  - if bit 4 is cleared, the macrotile is fast-cleared
    - the lower 4 bits encode the clear values (0 or 1) for color and alpha
    - for `VK_FORMAT_B8G8R8A8_UNORM` and `VK_FORMAT_D24_UNORM_S8_UINT`,
      - bit 0 is always 1
      - bit 1 is always 0
      - bit 2 is alpha value
      - bit 3 is color value

## Depth Formats

- vk has these formats
  - `VK_FORMAT_D16_UNORM`
  - `VK_FORMAT_X8_D24_UNORM_PACK32`
    - this is supported as z24s8
  - `VK_FORMAT_D32_SFLOAT`
  - `VK_FORMAT_S8_UINT`
  - `VK_FORMAT_D16_UNORM_S8_UINT`
    - this is unsupported
  - `VK_FORMAT_D24_UNORM_S8_UINT`
  - `VK_FORMAT_D32_SFLOAT_S8_UINT`
- depth/stencil buffer
  - `REG_A6XX_RB_DEPTH_BUFFER_INFO` specifies the depth/stencil buffer
    - `DEPTH6_16` is for `VK_FORMAT_D16_UNORM`
    - `DEPTH6_24_8` is for
      - `VK_FORMAT_X8_D24_UNORM_PACK32`
      - `VK_FORMAT_D24_UNORM_S8_UINT`
    - `DEPTH6_32` is for the rest
      - `VK_FORMAT_D32_SFLOAT`
      - `VK_FORMAT_D32_SFLOAT_S8_UINT`
      - `VK_FORMAT_S8_UINT`
        - why?
    - `DEPTH6_NONE` is no depth buffer
  - `REG_A6XX_RB_STENCIL_INFO` specifies the separate stencil buffer
    - this is for `VK_FORMAT_D32_SFLOAT_S8_UINT` or `VK_FORMAT_S8_UINT`
- to sample from the depth/stencil buffer,
  - vk can only sample either the depth or the stencil aspect
    - it returns `(D, 0, 0, 1)` or `(S, 0, 0, 1)`
    - if depth compare is enabled, `D` is replaced by `1.0` or `0.0`
  - `FMT6_16_UNORM` is for `VK_FORMAT_D16_UNORM`
  - `FMT6_32_FLOAT` is for
    - `VK_FORMAT_D32_SFLOAT`
    - `VK_FORMAT_D32_SFLOAT_S8_UINT` if depth aspect
  - `FMT6_8_UINT` is for
    - `VK_FORMAT_S8_UINT`
    - `VK_FORMAT_D32_SFLOAT_S8_UINT` if stencil aspect
  - `FMT6_Z24_UNORM_S8_UINT` is for
    - `VK_FORMAT_X8_D24_UNORM_PACK32`
    - `VK_FORMAT_D24_UNORM_S8_UINT` if depth aspect
    - I guess it returns `(D, 0, 0, 1)` despite the format name
      - `FMT6_Z24_UNORM_S8_UINT` (0xa0) used to be called
        `RB6_Z24_UNORM_S8_UINT` / `TFMT6_X8Z24_UNORM`.
  - `FMT6_Z24_UINT_S8_UINT` is for `VK_FORMAT_D24_UNORM_S8_UINT` if stencil
    aspect
    - this returns `(D, S, 0, 1)` and requires a swizzle to get `(S, 0, 0, 1)`
    - this is supported on gen2+
      - `FMT6_8_8_8_8_UINT` is used on gen1 and a swizzle is applied
      - because they are not ubwc-compatible, UBWC must be disabled
- image storage
  - it is similar to sampling, except the depth/stencil buffer can be
    loaded/stored
  - not all implementations support this
- event blit
  - this is the fast path to clear/load/store gmem when supported
  - `RB_BLIT_DST_INFO` specifies the format and the writemask (among others)
  - when clearing, the format is
    - `FMT6_16_UNORM`
    - `FMT6_32_FLOAT`
    - `FMT6_8_UINT`
    - `FMT6_Z24_UNORM_S8_UINT`
      - writemask controls d and/or s are cleared
  - when blitting, the format is almost the same as when sampling
    - except `FMT6_Z24_UNORM_S8_UINT_AS_R8G8B8A8` is used in place of
      `FMT6_Z24_UNORM_S8_UINT` and `FMT6_Z24_UINT_S8_UINT`
- 2d clear/blit
- 3d clear/blit

## LRZ

- depth test
  - depth test, or late-z, conceptually happens after fs
  - if fs does not modify the depth value, has no side effect, and does not
    kill, depth test can be performed before fs transparently
    - this is known as early-z and is huge because fs tends to be expensive
- lrz
  - lrz stands for low-resolution z
  - a value in lrz represents the minimum (or maximum) value for all values in
    the corresponding block in the original depth buffer
  - at binning pass, if enabled, lrz is updated and is used to speed up
    binning
  - at early-z, if enabled, lrz can be used as a pre-pass to reject blocks
    before using the full-resolution depth buffer
- depth image layout
  - depth buffer
  - lrz buffer
    - each pixel in the lrz buffer represents a 8x8 block in the original
      depth buffer
    - the format is always `VK_FORMAT_D16_UNORM`
  - fc buffer (always 512 bytes)
    - each bit in the fc buffer represents a 16x4 block in the lrz buffer
      - a value of 0 means the corresponding lrz block is fast-cleared (to 0.0
        or 1.0 depending on the direction)
      - a value of 1 means the corresponding lrz block is not fast-cleared
        (i.e., has been modified)
    - 512 bytes are enough for a 4096x4096 depth buffer
  - state tracking (always 6 bytes)
    - 1 byte for dir tracking
      - `CUR_DIR_DISABLED` (0x0): lrz is force-disabled because the dir is
        wrong
      - `CUR_DIR_GE` (0x1): the current direction is GE
      - `CUR_DIR_LE` (0x2): the current direction is LE
      - `CUR_DIR_UNSET` (0x3): `LRZ_CLEAR` event clears to this value, and the
        next access determines the current direction
    - 4 bytes for the depth view that lrz buffer is for
      - they are used to compare with `GRAS_LRZ_DEPTH_VIEW`, and if they
        differ, lrz is force-disabled
    - 1 byte for padding
- `GRAS_LRZ_CNTL`
  - `ENABLE` enables lrz in both binning and early-z
  - `LRZ_WRITE` enables lrz writes in the binning pass
  - `GREATER` updates max values instead of min values
  - `FC_ENABLE` enables fast-clear
    - this is always on in turnip unless hw limit is exceeded or `nolrzfc`
  - `Z_TEST_ENABLE`
  - `Z_BOUNDS_ENABLE`
  - `DIR` specifies the current dir
    - `LRZ_DIR_LE`
    - `LRZ_DIR_GE`
    - `LRZ_DIR_INVALID` (to invalidate the dir tracking byte)
  - `DIR_WRITE` updates the dir tracking byte
  - `DISABLE_ON_WRONG_DIR` force-disables lrz if the current dir differs from
    the dir tracking byte
    - this is always on on gen3+ in turnip
- lrz events
  - `LRZ_CLEAR` clears the fc buffer (when `FC_ENABLE`) and/or the dir
    tracking byte (when `DISABLE_ON_WRONG_DIR`) and/or the depth view bytes
  - `LRZ_FLUSH` flushes the lrz cache to the lrz buffer
- gen differences
  - turnip has 3 params
    - `enable_lrz_fast_clear` is set on all a6xx
    - `has_lrz_dir_tracking` is set on a6xx gen3+
    - `lrz_track_quirk` is set on a6xx gen3
  - fast-clear is available on all gens, but both turnip and blob use it on
    gen3+
  - on gen3 and only on gen3, `CP_REG_WRITE` with `TRACK_LRZ` must be used to
    write lrz registers
  - dir tracking is introduced to support lrz in secondary command buffers and
    dynamic rendering

## Misc Blocks

- RBBM: ring buffer status?
- DBGC: gpu status for debug?
- UCHE: unified cache / L2 (read-write)
- TPL1: texture process / L1 (read-only)
- CCU: color cache unit (read-write L1?)

## Performance Counters

- how does it work
  - there appears to be a RBBM timer that runs at 19.2MHz
  - at each cycle, RBBM samples the selected events and increments the
    corresponding counter registers
  - `VFD_PERFCTR_VFD_SEL(i)`, `SP_PERFCTR_SP_SEL(i)`, `RB_PERFCTR_RB_SEL(i)`,
    etc are used to select the events to sample
    - for each block, there are N events and M select registers, with N >= M
  - `RBBM_PERFCTR_VFD(i)`, `RBBM_PERFCTR_SP(i)`, `RBBM_PERFCTR_RB(i)`, etc.
    are free-running counters
    - RBBM increments these counter registers after sampling the events
      selected by the select registers
    - for each block, there are M select registers and M counter registers in
      RBBM
  - these registers are represented by `fd_perfcntr_counter` in turnip
    - the selected events are `fd_perfcntr_countable`
- for example,
  - `CP_ALWAYS_ON_COUNTER` is a register that increments at gpu freq
  - we can set `CP_PERFCTR_CP_SEL(i)` to select `PERF_CP_ALWAYS_COUNT` event
  - every cycle of RBBM (at 19.2MHz), it adds the delta of
    `CP_ALWAYS_ON_COUNTER` to `RBBM_PERFCTR_CP(i)`
  - this gives us gpu freq
- `fdperf`
  - some counters are reserved for the kernel and cannot be used
  - `fdperf` also reserves a CP counter internally for `PERF_CP_ALWAYS_COUNT`,
    to display the GPU frequency
  - it resamples every 500ms, `REFRESH_MS`
  - it always takes the delta of the counter values between the last two
    samples and normalize them to be over 1 second
  - if a selected event has `CYCLE`, `BUSY`, or `IDLE` in its name, it divides
    the normalized delta by max freq and displays as percentage
  - otherwise, it displays the normalized delta
- CP
  - `PERF_CP_ALWAYS_COUNT`
    - count gpu cycles
    - used to get gpu freq
  - `PERF_CP_BUSY_GFX_CORE_IDLE`
    - count cycles where CP is busy and core is idle
    - if high, CP-bound
  - `PERF_CP_BUSY_CYCLES`
    - count cycles where CP is busy
  - `PERF_CP_NUM_PREEMPTIONS`
  - `PERF_CP_PREEMPTION_REACTION_DELAY`
  - `PERF_CP_PREEMPTION_SWITCH_OUT_TIME`
  - `PERF_CP_PREEMPTION_SWITCH_IN_TIME`
  - `PERF_CP_DEAD_DRAWS_IN_BIN_RENDER`
  - `PERF_CP_PREDICATED_DRAWS_KILLED`
  - `PERF_CP_MODE_SWITCH`
  - `PERF_CP_ZPASS_DONE`
    - count fragments passing z-test
  - `PERF_CP_CONTEXT_DONE`
  - `PERF_CP_CACHE_FLUSH`
- VFD
  - `PERF_VFD_BUSY_CYCLES`
    - count cycles where VFD is busy
    - when VFD is stalled or starved (thus not doing real work), it still
      counts as busy
  - `PERF_VFD_STALL_CYCLES_UCHE`
    - count cycles where VFD is stalled because UCHE write is not ready
  - `PERF_VFD_STALL_CYCLES_VPC_ALLOC`
  - `PERF_VFD_STALL_CYCLES_SP_INFO`
  - `PERF_VFD_STALL_CYCLES_SP_ATTR`
  - `PERF_VFD_STARVE_CYCLES_UCHE`
    - count cycles where VFD is starved because UCHE read is not ready
  - `PERF_VFD_RBUFFER_FULL`
  - `PERF_VFD_ATTR_INFO_FIFO_FULL`
- VPC
  - `PERF_VPC_BUSY_CYCLES`
    - count cycles where VFD is busy
  - `PERF_VPC_WORKING_CYCLES`
    - count cycles where VFD is busy and doing real work
    - that is, not stalled or starved
  - `PERF_VPC_STALL_CYCLES_UCHE`
  - `PERF_VPC_STALL_CYCLES_VFD_WACK`
  - `PERF_VPC_STALL_CYCLES_HLSQ_PRIM_ALLOC`
  - `PERF_VPC_STALL_CYCLES_PC`
- PC
  - `PERF_PC_BUSY_CYCLES`
  - `PERF_PC_WORKING_CYCLES`
  - `PERF_PC_STALL_CYCLES_VFD`
  - `PERF_PC_STALL_CYCLES_TSE`
  - `PERF_PC_STALL_CYCLES_VPC`
  - `PERF_PC_STALL_CYCLES_UCHE`
  - `PERF_PC_STALL_CYCLES_TESS`
  - `PERF_PC_STALL_CYCLES_TSE_ONLY`
- TSE
  - `PERF_TSE_BUSY_CYCLES`
  - `PERF_TSE_CLIPPING_CYCLES`
  - `PERF_TSE_STALL_CYCLES_RAS`
  - `PERF_TSE_STALL_CYCLES_LRZ_BARYPLANE`
- VSC
  - `PERF_VSC_BUSY_CYCLES`
  - `PERF_VSC_WORKING_CYCLES`
- RAS
  - `PERF_RAS_BUSY_CYCLES`
  - `PERF_RAS_SUPERTILE_ACTIVE_CYCLES`
  - `PERF_RAS_STALL_CYCLES_LRZ`
  - `PERF_RAS_STARVE_CYCLES_TSE`
- LRZ
  - `PERF_LRZ_BUSY_CYCLES`
  - `PERF_LRZ_STARVE_CYCLES_RAS`
  - `PERF_LRZ_STALL_CYCLES_RB`
  - `PERF_LRZ_STALL_CYCLES_VSC`
- RB
  - `PERF_RB_BUSY_CYCLES`
  - `PERF_RB_STALL_CYCLES_HLSQ`
  - `PERF_RB_STALL_CYCLES_FIFO0_FULL`
  - `PERF_RB_STALL_CYCLES_FIFO1_FULL`
  - `PERF_RB_STALL_CYCLES_FIFO2_FULL`
  - `PERF_RB_STARVE_CYCLES_SP`
  - `PERF_RB_STARVE_CYCLES_LRZ_TILE`
  - `PERF_RB_STARVE_CYCLES_CCU`
- CCU
  - `PERF_CCU_BUSY_CYCLES`
  - `PERF_CCU_STALL_CYCLES_RB_DEPTH_RETURN`
  - `PERF_CCU_STALL_CYCLES_RB_COLOR_RETURN`
  - `PERF_CCU_STARVE_CYCLES_FLAG_RETURN`
  - `PERF_CCU_DEPTH_BLOCKS`
- SP
  - `PERF_SP_BUSY_CYCLES`
  - `PERF_SP_ALU_WORKING_CYCLES`
  - `PERF_SP_EFU_WORKING_CYCLES`
  - `PERF_SP_STALL_CYCLES_VPC`
  - `PERF_SP_STALL_CYCLES_TP`
  - `PERF_SP_STALL_CYCLES_UCHE`
  - `PERF_SP_STALL_CYCLES_RB`
  - `PERF_SP_NON_EXECUTION_CYCLES`
  - `PERF_SP_WAVE_CONTEXTS`
  - `PERF_SP_WAVE_CONTEXT_CYCLES`
  - `PERF_SP_FS_STAGE_WAVE_CYCLES`
  - `PERF_SP_FS_STAGE_WAVE_SAMPLES`
  - `PERF_SP_VS_STAGE_WAVE_CYCLES`
  - `PERF_SP_VS_STAGE_WAVE_SAMPLES`
  - `PERF_SP_FS_STAGE_DURATION_CYCLES`
  - `PERF_SP_VS_STAGE_DURATION_CYCLES`
  - `PERF_SP_WAVE_CTRL_CYCLES`
  - `PERF_SP_WAVE_LOAD_CYCLES`
  - `PERF_SP_WAVE_EMIT_CYCLES`
  - `PERF_SP_WAVE_NOP_CYCLES`
  - `PERF_SP_WAVE_WAIT_CYCLES`
  - `PERF_SP_WAVE_FETCH_CYCLES`
  - `PERF_SP_WAVE_IDLE_CYCLES`
- HLSQ
  - `PERF_HLSQ_BUSY_CYCLES`
  - `PERF_HLSQ_STALL_CYCLES_UCHE`
  - `PERF_HLSQ_STALL_CYCLES_SP_STATE`
  - `PERF_HLSQ_STALL_CYCLES_SP_FS_STAGE`
  - `PERF_HLSQ_UCHE_LATENCY_CYCLES`
  - `PERF_HLSQ_UCHE_LATENCY_COUNT`
- TP
  - `PERF_TP_BUSY_CYCLES`
  - `PERF_TP_STALL_CYCLES_UCHE`
  - `PERF_TP_LATENCY_CYCLES`
  - `PERF_TP_LATENCY_TRANS`
  - `PERF_TP_FLAG_CACHE_REQUEST_SAMPLES`
  - `PERF_TP_FLAG_CACHE_REQUEST_LATENCY`
  - `PERF_TP_L1_CACHELINE_REQUESTS`
  - `PERF_TP_L1_CACHELINE_MISSES`
  - `PERF_TP_SP_TP_TRANS`
  - `PERF_TP_TP_SP_TRANS`
  - `PERF_TP_OUTPUT_PIXELS`
  - `PERF_TP_FILTER_WORKLOAD_16BIT`
- UCHE
  - `PERF_UCHE_BUSY_CYCLES`
  - `PERF_UCHE_STALL_CYCLES_ARBITER`
  - `PERF_UCHE_VBIF_LATENCY_CYCLES`
  - `PERF_UCHE_VBIF_LATENCY_SAMPLES`
  - `PERF_UCHE_VBIF_READ_BEATS_TP`
  - `PERF_UCHE_VBIF_READ_BEATS_VFD`
  - `PERF_UCHE_VBIF_READ_BEATS_HLSQ`
  - `PERF_UCHE_VBIF_READ_BEATS_LRZ`
  - `PERF_UCHE_VBIF_READ_BEATS_SP`
  - `PERF_UCHE_READ_REQUESTS_TP`
  - `PERF_UCHE_READ_REQUESTS_VFD`
  - `PERF_UCHE_READ_REQUESTS_HLSQ`

## ISA

- an instruction has 64 bits
- gpr register number has 6 bits
  - r0..r47 are general registers
  - r48..r55 are shared registers
  - r61 is address register (a0)
  - r62 is predicate register (p0)
  - half registers (hr0..hr47) alias the general registers
- const register number has 9 bits
  - c0..c511 are const registers
- an SP can have up to 16 waves
  - wave size can be 64 or 128
  - getspid, getwid, getfiberid
- register files
  - 128KB GPR
  - `ir3_pressure` used in `ir3_ra` has 16-bit words as units
  - `RA_HALF_SIZE` is `4 * 48`, because there are hr0..hr47
  - `RA_FULL_SIZE` is `4 * 48 * 2`, because there are r0..r47
  - `RA_SHARED_SIZE` is `2 * 4 * 8` because there are r48..r55
- 32KB local memory
  - as defined in OpenCL
- category 0
  - flow control, predication, prologue
- category 1
  - move, swizzle, gather/scatter
- category 2
  - 1-/2-src ALUs
- category 3
  - 3-src ALUs
- category 4
  - EFU (elementary function unit), aka SFU (special function unit), aka math
- category 5
  - texture
- category 6
  - load/store/atomic
- category 7
  - barrier
- instructions
  - `shps` prologue starts
  - `shpe` prologue ends
  - `getone` branches all but one fiber
  - `movmsk` sets all bits to 1?
  - `shl.b` shifts left
  - `cmps.s` compares signed
  - `and.b` logical AND bool
  - `br` branches
  - `isam` samples
  - `stc` stores const (register file)
  - `stib` stores ibo/ssbo (memory)
  - `stl` stores local (memory)
  - `stg` stores global (memory)
  - `chmask` resets execution mask
  - `chsh` jumps to the next shader
    - `cmask` and `chsh` are used by VS to chain to HS or GS
    - VS should also do `stl` outputs to local storage, for use by HS or GS
- flags
  - `IR3_INSTR_SY` is set after sample instructions to avoid data hazard
  - `IR3_INSTR_SS` is set after long (EFU/math) instructions to avoid data
    hazard
  - `IR3_INSTR_JP` is set on jump targets.. why?
  - `IR3_INSTR_UL` is set on the last use of `a0.x`... why?
  - `IR3_INSTR_3D` is set when sampling 3D images
  - `IR3_INSTR_A` is set when sampling array images
  - `IR3_INSTR_O` is set when sampling with offset
  - `IR3_INSTR_P` is set when sampling with projection
  - `IR3_INSTR_S` is set when sampling with shadow comparison
  - `IR3_INSTR_S2EN` is set when sampl/tex idx is in another register
  - `IR3_INSTR_SAT` saturates the result
  - `IR3_INSTR_B` if the resource is bindless
  - these are not translated directly to hw encoding
    - `IR3_INSTR_NONUNIF` if res idx is non-uniform
    - `IR3_INSTR_A1EN` uses `a1.x`
    - `IR3_INSTR_MARK`
    - `IR3_INSTR_UNUSED`

## Reverse Engineering

- <https://gitlab.freedesktop.org/freedreno/freedreno.git>
  - with <https://gitlab.freedesktop.org/freedreno/freedreno/-/merge_requests/19>
  - `BUILD=ndk NDK_PATH=~/android/sdk/ndk/25.0.8775105 make libwrap.so libwrapfake.so`
- <https://github.com/olvaffe/vktest.git>
  - edit NDK path in `machines/android-arm64.txt`
  - `meson --cross-file machines/android-arm64.txt out-android`
- using `libwrap.so`
  - `adb push libwrap.so /data/local/tmp` and set
    `LD_PRELOAD=/data/local/tmp/libwrap.so`
    - it will intercept ioctls to kgls
  - ioctls are dumped to stdout and rd is saved to `/sdcard/trace.rd`
- using `libwrapfake.so`
  - additionally set `WRAP_GPU_ID=635.0` and `WRAP_GMEM_SIZE=0x80000`
- <https://gitlab.freedesktop.org/freedreno/freedreno/-/wikis/Reverse-Engineering-Tools/How-To-Fake-GPU-With-Chroot-On-Android-11>
  - search <https://vulkan.gpuinfo.org/> to find the device with the latest
    driver
  - find the latest recovery image for the device
  - unpack images
    - `brotli -d vendor.new.dat.br`
    - `sdat2img.py vendor.transfer.list vendor.new.dat vendor.img`
    - `ext2rd vendor.img ./:vendor`
  - chroot
    - `adb root`
    - `adb shell mkdir /data/chroot`
    - `sudo adb push vendor /data/chroot`
    - `sudo adb push system/* /data/chroot`

## Display Pipeline

- SSPP: Source Surface Processor Pipes
  - read layers from memory
  - limited pre-blending per-layer processing
    - chroma upsampling
  - `sde_hw_sspp.h`
  - scaler
  - QoS?
  - sharpen
  - multirect
  - solidfill
  - csc (yuv->rgb)
  - format
    - rot90, flip h/v
    - chroma upsampling
    - deinterleave
    - ubwc
  - placeholders
    - pixel extension: hue, sat, val, cont, memcolor
    - histogram
    - igc (inverse gamma correction)
- layer mixer
  - `sde_hw_lm.h`
  - misr?
  - dimlayer?
  - bordercolor
  - alpha-out (select fg or bg)
  - blend-config (fg alpha, bg alpha, blend op)
  - mixer-out (set output size)
  - placeholders
    - gc (gamma correction)
- DSPP: Destination Surface Processor Pipes
  - post-blending global processing
  - `sde_hw_dspp.h`
  - `msm_drm_pp.h`
  - igc - inverse gamma correction
  - pcc - panel color correction (3x3 / 3x11 matrix, polynomial)
  - gc - gamma correction
  - pa hsic - hue, sat, val, cont
  - memcolor - skin, ski, foliage, protect
  - six-zone?
  - gamut - 3D lut, size 17 or 5 or 13
  - dither - 4x4 matrix, strength, etc.
  - histogram
  - vlut? - size 256
  - ad?
  - placeholders
    - sharpening
    - danger-safe

## vsync

- Example timing for 2560x1440
  - clock 237800 khz
  - hdisplay 2560
  - hsync start 2608 (front porch 48)
  - hsync end 2640 (hblank 32)
  - htotal 2720 (back porch 80)
  - vdisplay 1440
  - vsync start 1442 (front porch 2)
  - vsync end 1445 (vblank 3)
  - vtotal 1457 (back porch 12)
  - vrefresh 60
  - hblank 32/clock = 0.1us
  - vblank 3*2720/clock = 34.3us
  - v-inactive 17*2720/clock = 0.2ms

- IRQs per frame
  - ping-pong done irq
    - shortly before vsync (~2.2ms)
    - double-buffered registers for the next frame latched
    - non-double-buffered registers for the next frame can be programmed
  - vsync irq
    - new frame start / vblank end
    - current frame end / vblank start
    - unclear which
    - the previous buffer can be modified

- ATOMIC UPDATE
  `mdss_mdp_overlay_kickoff`, 3ms
   - `sspp_programming`, 0.3ms
     - map buffers
   - `dest_scaler_programming`, 0.1ms
   - `display_commit`, 2.5ms
     - `prepare_fnc`, 1us
     - `mixer_programming`, 0.1ms
     - `postproc_programming`, 2us
     - `frame_ready`
     - `wait_pingpong`, 2~6ms
       - ping pong done interrupt indicates the previous mdp kickoff has
         completed
       - happens ~2ms before vsync
       - wait for it before updating non-double-buffered registers
     - `flush_kickoff`, 0.1ms
       - flush register changes
   - `display_wait4comp`, 0.02ms
     - busy wait for register flush to complete
   - `overlay_cleanup`, 0.1ms
     - unmap buffers

- `mdss_fb_0`
  - incremented after `wait_pingpong` and before `flush_kickoff`
  - basically, incremented after each ping-pong IRQ
- `mdss_fb0_retire`
  - incremented after each vsync irq
