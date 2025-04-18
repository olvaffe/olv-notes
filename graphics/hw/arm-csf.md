ARM Mali CSF
============

## HW

- MCU
  - it shares the system memory with cpu for FW, GLB, CSGs, and CSs
  - it executes the FW and exposes GLB/CSGs/CSs to host driver
  - it supports multiple CSGs and schedules CSs to CSHWIFs
  - it emulates some of the CSF instructions
- CSHW
  - it has two or more CSHWIFs to execute CSs concurrently
    - each CSHWIF has its own register file and scoreboard
  - some CSF instrs are executed by CEU
  - some CSF instrs are emulated by MCU
- hardware iterators
  - they take jobs from CSHW, split jobs into tasks, and dispatch tasks to
    endpoints (tiler or shader cores depending on the tasks)
- registers
  - `GPU_CONTROL` regs for gpu control
  - `JOB_CONTROL` regs for csf-to-host irq control
  - `DOORBELLS` regs for host-to-csf signaling
  - `USER` regs for direct userspace access

## `GPU_CONTROL` regs

- `GPU_ID`
  - arch major/minor/rev, where arch major/minor identifies the fw to load
  - prod major identifies the product line for the arch major
  - ver major/minor/status
- `L2_FEATURES` (`GPU_L2_FEATURES` in panthor)
  - bit 7:0 (`GPU_L2_FEATURES_LINE_SIZE`): log2 of cache line size (e.g., 6
    indicates 64)
  - bit 15:8: log2 of cache associativity (e.g., 3 indicates 8)
  - bit 23:16: log2 of cache slice size (e.g., 0x12 indicates 256KB per L2 slice)
  - bit 31:24: log2 of bus width in bits (e.g., 7 indicates 128)
- `CORE_FEATURES` (`GPU_CORE_FEATURES` in panthor)
- `TILER_FEATURES` (`GPU_TILER_FEATURES` in panthor)
  - bit 5:0: log2 of bin size (e.g., 9 indicates 512 bytes)
  - bit 11:8: max level (e.g., 8, but should use hardcoded 4 on later archs)
- `MEM_FEATURES` (`GPU_MEM_FEATURES` in panthor)
  - bit 0 (`GROUPS_L2_COHERENT`) indicates shader cores within a group are
    coherent
  - bit 11:8: L2 slice count minus 1 (e.g., 3 indicates 4 L2 slices)
- `MMU_FEATURES` (`GPU_MMU_FEATURES` in panthor)
  - bit 7:0: va bits (e.g., 48)
  - bit 15:8: pa bits (e.g., 40)
- `AS_PRESENT` (`GPU_AS_PRESENT` in panthor)
  - e.g., 0xff indicates AS 0 to 7
- `CSF_ID` (`GPU_CSF_ID` in panthor)
- `GPU_IRQ_RAWSTAT` (`GPU_INT_RAWSTAT` in panthor)
  - irq sources that are active
  - bit 0 (`GPU_IRQ_FAULT`)
  - bit 1 (`GPU_IRQ_PROTM_FAULT`)
  - bit 8 (`GPU_IRQ_RESET_COMPLETED`)
  - bit 17 (`GPU_IRQ_CLEAN_CACHES_COMPLETED`)
- `GPU_IRQ_CLEAR` (`GPU_INT_CLEAR` in panthor)
  - irq sources to clear
- `GPU_IRQ_MASK` (`GPU_INT_MASK` in panthor)
  - irq sources to enable
- `GPU_IRQ_STATUS` (`GPU_INT_STAT` in panthor)
  - `GPU_IRQ_RAWSTAT & GPU_IRQ_MASK`
- `GPU_COMMAND` (`GPU_CMD` in panthor)
  - `GPU_SOFT_RESET` resets gpu and leaves external bus in defined and idle state
  - `GPU_FLUSH_CACHES` flushes/invalidtes L2/LSC/other caches
- `GPU_STATUS`
- `GPU_FAULTSTATUS` (`GPU_FAULT_STATUS` in panthor)
  - bit 7:0: exception type
    - `GPU_BUS_FAULT` indicates external bus fault
    - `GPU_SHAREABILITY_FAULT` indicates a cache line is accessed as both
      shareable and non-shareable within gpu
    - `SYSTEM_SHAREABILITY_FAULT` indicates a cache line is accessed as both
      shareable and non-shareable outside gpu
    - `GPU_CACHEABILITY_FAULT` indicates a cache line is accessed as both
      cacheable and non-cacheable within gpu
  - bit 9:8: access type (r, w, x)
  - bit 10: `GPU_FAULTADDRESS` is valid
  - bit 11: job AS id is valid
  - bit 15:12: job AS id
  - bit 31:16: source id
- `GPU_FAULTADDRESS` (`GPU_FAULT_ADDR_LO` in panthor)
- `L2_CONFIG`
- `PWR_KEY` (`GPU_PWR_KEY` in panthor)
- `PWR_OVERRIDE0` (`GPU_PWR_OVERRIDE0` in panthor)
- `PWR_OVERRIDE1` (`GPU_PWR_OVERRIDE1` in panthor)
- `GPU_FEATURES`
- `PRFCNT_FEATURES`
- `TIMESTAMP_OFFSET` (`GPU_TIMESTAMP_OFFSET_LO` in panthor)
- `CYCLE_COUNT` (`GPU_CYCLE_COUNT_LO` in panthor)
- `TIMESTAMP` (`GPU_TIMESTAMP_LO` in panthor)
  - returns the current value of the system counter whose frequency can be
    read from `CNTFRQ_EL0`
- `THREAD_MAX_THREADS` (`GPU_THREAD_MAX_THREADS` in panthor)
- `THREAD_MAX_WORKGROUP_SIZE` (`GPU_THREAD_MAX_WORKGROUP_SIZE` in panthor)
- `THREAD_MAX_BARRIER_SIZE` (`GPU_THREAD_MAX_BARRIER_SIZE` in panthor)
- `THREAD_FEATURES` (`GPU_THREAD_FEATURES` in panthor)
- `TEXTURE_FEATURES` (`GPU_TEXTURE_FEATURES` in panthor)
  - supported compressed formats
- `TEXTURE_FEATURES_1`
- `TEXTURE_FEATURES_2`
- `TEXTURE_FEATURES_3`
- `DOORBELL_FEATURES`
- `GPU_COMMAND_ARG0`
- `GPU_COMMAND_ARG1`
- `SHADER_PRESENT` (`GPU_SHADER_PRESENT_LO` in panthor)
  - e.g., 0x50005 indicates shader core 0, 2, 16, 18
- `TILER_PRESENT` (`GPU_TILER_PRESENT_LO` in panthor)
  - e.g., 0x1 indicates tiler 0
- `L2_PRESENT` (`GPU_L2_PRESENT_LO` in panthor)
  - e.g., 0x1 indicates L2 0
- `SHADER_READY`
- `TILER_READY`
- `L2_READY`
- `SHADER_PWRON`
- `SHADER_PWRFEATURES`
- `TILER_PWRON`
- `L2_PWRON`
- `SHADER_PWROFF`
- `TILER_PWROFF`
- `L2_PWROFF`
- `SHADER_PWRTRANS`
- `TILER_PWRTRANS`
- `L2_PWRTRANS`
- `SHADER_PWRACTIVE`
- `TILER_PWRACTIVE`
- `L2_PWRACTIVE`
  - ok, this works the same way for `SHADER`, `TILER`, and `L2` blocks
    - but it seems `SHADER` and `TILER` are controlled by CSF
    - only `L2` is controlled by the host driver
  - `L2_READY` indicates L2 is powered up and ready
  - `L2_PWRON` requests L2 to be powered on
  - `L2_PWROFF` requests L2 to be powered off
  - `L2_PWRTRANS` indicates L2 is undergone power transition
  - `L2_PWRACTIVE` indicates L2 is busy
- `REVIDR` (`GPU_REVID` in panthor)
- `ASN_HASH`
- `AMBA_FEATURES` (`GPU_COHERENCY_FEATURES` in panthor)
  - indicates protocols/features supported by the amba bus the gpu is
    connected to
- `AMBA_ENABLE` (`GPU_COHERENCY_PROTOCOL` in panthor)
  - selects protocols/features of the amba bus the gpu is connected to
- `SYSC_PBHA_OVERRIDE`
- `SYSC_ALLOC`
- `MCU_CONTROL`
  - `DISABLE` requests mcu to stop
  - `ENABLE` requests mcu to start
  - `AUTO` requests mcu to start, and will restart automatically on fast reset
- `MCU_STATUS`
  - `DISABLED` means mcu is not running
  - `ENABLED` means mcu is running
  - `HALT` means mcu halts itself (for fast reset)
  - `FATAL` means mcu hits fatal error
- `MCU_FEATURES`

## `MMU_CONTROL` regs

- MMU is not a part of CSF
- `IRQ_RAWSTAT` (`MMU_INT_RAWSTAT` in panthor)
  - irq sources that are active
  - bit 15:0: which AS page faults
  - bit 31:16: which AS has cmd completed
- `IRQ_CLEAR` (`MMU_INT_CLEAR` in panthor)
  - irq sources to clear
- `IRQ_MASK` (`MMU_INT_MASK` in panthor)
  - irq sources to enable
- `IRQ_STATUS` (`MMU_INT_STAT` in panthor)
  - `IRQ_RAWSTAT & IRQ_MASK`
- `ASn` (`MMU_AS` in panthor)
  - `TRANSTAB` is the addr of the base page table
  - `MEMATTR` is the memattrs for 8 memory types
    - bit 0: alloc cacheline for inner cache write
    - bit 1: alloc cacheline for inner cache read
    - bit 3:2: respect bit 0 and 1, or let bus decide
    - bit 5:4: coherency
    - bit 7:6: shared, non-cached, write-back, fault
  - `LOCKADDR` specifies addr/size for command
  - `COMMAND`
    - `AS_COMMAND_NOP` is nop
    - `AS_COMMAND_UPDATE` commits `TRANSTAB`, `MEMATTR`, and `TRANSCFG` to hw
    - `AS_COMMAND_LOCK` locks a region for pt update
    - `AS_COMMAND_UNLOCK` unlocks a region
    - `AS_COMMAND_FLUSH_PT` flushes/invalidates L2 and then unlock
    - `AS_COMMAND_FLUSH_MEM` flushes/invalidates all caches and then unlock
  - `FAULTSTATUS`
    - bit 7:0: exception type
      - `TRANSLATION_FAULT_n` hits an invalid pt entry at level n
      - `PERMISSION_FAULT_n` violates permission, level n
      - `ACCESS_FLAG_n` violates page table access flag
      - `ADDRESS_SIZE_FAULT_IN` violates `TRANSCFG` input addr bits
      - `ADDRESS_SIZE_FAULT_OUTn` violates `TRANSCFG` output addr bits
      - `MEMORY_ATTRIBUTE_FAULT_n` means `MEMATTR` level n disallows access
    - bit 9:8: access type (r, w, x, atomic)
    - bit 31:16: source id
  - `FAULTADDRESS`
  - `STATUS`
    - bit 0: an external command is active
    - bit 1: an internal command is active
  - `TRANSCFG`
    - bit 3:0: `AS_TRANSCFG_ADRMODE_AARCH64_4K` means 4K page table
    - bit 10:6: 55 minus input address bits (e.g., 7 for 48-bit)
    - bit 18:14: 55 minus output address bits, but can be 0
    - bit 22: `AS_TRANSCFG_SL_CONCAT`
    - bit 25:24: NC, WB, etc.
    - bit 29:28: non-shareable, outer-shareable, inner-shareable
    - bit 30: `AS_TRANSCFG_PTW_RA` read should allocate cacheline
    - bit 33: disable hierarchical access perm
    - bit 34: disable access fault checking
    - bit 35: disallow exec on writable page
    - bit 36: force exec on readable page
  - `FAULTEXTRA` is extra fault info

## `JOB_CONTROL` regs

- `JOB_IRQ_RAWSTAT` (`JOB_INT_RAWSTAT` in panthor)
  - irq sources that are active
  - bit 30:0: CSGn
  - bit 31: GLB
- `JOB_IRQ_CLEAR` (`JOB_INT_CLEAR` in panthor)
  - irq sources to clear
- `JOB_IRQ_MASK` (`JOB_INT_MASK` in panthor)
  - irq sources to enable
- `JOB_IRQ_STATUS` (`JOB_INT_STATUS` in panthor)
  - `JOB_IRQ_RAWSTAT & JOB_IRQ_MASK`

## `DOORBELLS` regs

- `DOORBELLn` (`CSF_DOORBELL` in panthor)
  - `DOORBELL` write 1 to ring

## `USER` regs

- `LATEST_FLUSH` (`CSF_GPU_LATEST_FLUSH_ID` in panthor)
  - bit 23:0: flush id, incremented by full gpu cache flush/invalidate

## `GLB_CONTROL_BLOCK` regs

- these are virtual regs
  - `panthor_fw_global_control_iface` in panthor
- `GLB_VERSION` major/minor/patch version (e.g., 1.1.0)
- `GLB_FEATURES`
- `GLB_INPUT_VA` va of `GLB_INPUT_BLOCK` regs
- `GLB_OUTPUT_VA` va of `GLB_OUTPUT_BLOCK` regs
- `GLB_GROUP_NUM` number of `GROUP_CONTROL_BLOCK` instances
- `GLB_GROUP_STRIDE` stride between `GROUP_CONTROL_BLOCK` instances
- `GLB_PRFCNT_SIZE`
- `GLB_INSTR_FEATURES`
- `GLB_PRFCNT_FEATURES`

## `GLB_INPUT_BLOCK` regs
- these are virtual regs
  - `panthor_fw_global_input_iface` in panthor
- `GLB_REQ`
  - `HALT` halts mcu, in prep for fast reset
  - `CFG_PROGRESS_TIMER` commits `GLB_PROGRESS_TIMER`
  - `CFG_ALLOC_EN` commits `GLB_ALLOC_EN`
  - `CFG_PWROFF_TIMER` commits `GLB_PWROFF_TIMER`
  - `PROTM_ENTER` enters protected mode
  - `PRFCNT_ENABLE`
  - `PRFCNT_SAMPLE`
  - `COUNTER_ENABLE`
  - `PING` pings the mcu
  - `FIRMWARE_CONFIG_UPDATE`
  - `IDLE_ENABLE` enables/disables idle timer
  - `ITER_TRACE_ENABLE`
  - `SLEEP`
  - `INACTIVE_COMPUTE`
  - `INACTIVE_FRAGMENT`
  - `INACTIVE_TILER`
  - `PROTM_EXIT` exits protected mode
  - `PRFCNT_THRESHOLD`
  - `PRFCNT_OVERFLOW`
  - `IDLE_EVENT` acks idle event
- `GLB_ACK_IRQ_MASK`
  - irq sources to enable
  - when `GLB_ACK` is updated, and the changed bits are in `GLB_ACK_IRQ_MASK`,
    generate an irq
- `GLB_DB_REQ` rings the doorbell of CSGn
- `GLB_PROGRESS_TIMER` if a task runs too long, exceeding the specified
  cycles, generates an irq
- `GLB_PWROFF_TIMER` automatic PM for tiler and shader cores
  - bit 30:0: timeout val
  - bit 31: `GPU_COUNTER` when set, `SYSTEM_TIMESTAMP` when cleared
  - when non-zero,
    - it powers on a tile or shader core when there is a task
    - it powers down a tile or shader core after idling for timeout
    - it replaces manual `SHADER_PWRON` and `TILER_PWRON` control
- `GLB_ALLOC_EN` which shader cores are enabled
- `GLB_PRFCNT_JASID`
- `GLB_PRFCNT_BASE`
- `GLB_PRFCNT_EXTRACT`
- `GLB_PRFCNT_CONFIG`
- `GLB_PRFCNT_CSG_SELECT`
- `GLB_PRFCNT_FW_EN`
- `GLB_PRFCNT_CSG_EN`
- `GLB_PRFCNT_CSF_EN`
- `GLB_PRFCNT_SHADER_EN`
- `GLB_PRFCNT_TILER_EN`
- `GLB_PRFCNT_MMU_L2_EN`
- `GLB_IDLE_TIMER` if the gpu is idle for this timeout, set `IDLE_EVENT` in `GLB_ACK`
  - when there are more groups than CSGs, this can be used to rotate groups
- `GLB_IDLE_TIMER_CONFIG`
- `GLB_PWROFF_TIMER_CONFIG`
- `GLB_DEBUG_ARG_INn`
- `GLB_DEBUG_ACK_IRQ_MASK`
- `GLB_DEBUG_REQ`

## `GLB_OUTPUT_BLOCK` regs

- these are virtual regs
  - `panthor_fw_global_output_iface` in panthor
- `GLB_ACK` see `GLB_REQ`
- `GLB_DB_ACK` see `GLB_DB_REQ`
- `GLB_FATAL_STATUS`
- `GLB_PRFCNT_STATUS`
- `GLB_PRFCNT_INSERT`
- `GLB_DEBUG_ARG_OUTn`
- `GLB_DEBUG_ACK`

## `GROUP_CONTROL_BLOCK` regs

- these are virtual regs
  - `panthor_fw_csg_control_iface` in panthor
- `GROUP_FEATURES`
- `GROUP_INPUT_VA` va of `CSG_INPUT_BLOCK`
- `GROUP_OUTPUT_VA` va of `CSG_OUTPUT_BLOCK`
- `GROUP_SUSPEND_SIZE` size needed to suspend a CSG
- `GROUP_PROTM_SUSPEND_SIZE` size needed to suspend a CSG in protected mode
- `GROUP_STREAM_NUM` number of `STREAM_CONTROL_BLOCK` instances
- `GROUP_STREAM_STRIDE` stride between `STREAM_CONTROL_BLOCK` instances

## `CSG_INPUT_BLOCK` regs

- these are virtual regs
  - `panthor_fw_csg_input_iface` in panthor
- `CSG_REQ`
  - `STATE` requests to terminate/start/suspend/resume a CSG
  - `EP_CFG` commits `CSG_EP_REQ` and `CSG_ALLOW_*`
  - `STATUS_UPDATE` requuests to update `CSG_STATUS_*` and `CS_STATUS_*`
  - `SYNC_UPDATE` acks `SYNC_UPDATE`
    - gpu sets `SYNC_UPDATE` in `CSG_ACK` when the CSG executes certain `SYNC_*`
      instructions (i guess when the sync scope is system)
  - `IDLE` acks `IDLE`
    - gpu sets `IDLE` in `CSG_ACK` when the CSG becomes idle
  - `PROGRESS_TIMER_EVENT` acks `PROGRESS_TIMER_EVENT`
- `CSG_ACK_IRQ_MASK`
  - irq sources to enable
  - when `CSG_ACK` is updated, and the changed bits are in `CSG_ACK_IRQ_MASK`,
    generate an irq
- `CSG_DB_REQ` rings the doorbell of CSn
- `CSG_IRQ_ACK` acks irq for a CSn
- `CSG_ALLOW_COMPUTE` enables specified shader cores for compute
- `CSG_ALLOW_FRAGMENT` enables specified shader cores for frag
- `CSG_ALLOW_OTHER` enables specified tilers
- `CSG_EP_REQ`
  - `COMPUTE_EP` max shader cores for compute
  - `FRAGMENT_EP` max shader cores for frag
  - `TILER_EP` max tilers
  - `EXCLUSIVE_COMPUTE`
  - `EXCLUSIVE_FRAGMENT`
  - `PRIORITY` 0..15, with 15 being the highest priority
- `CSG_SUSPEND_BUF` va of the suspend buf
- `CSG_PROTM_SUSPEND_BUF` va of the suspend buf in protected mode
- `CSG_CONFIG`
  - bit 3:0: ASn
- `CSG_ITER_TRACE_CONFIG`
- `CSG_DVS_BUF`

## `CSG_OUTPUT_BLOCK` regs

- these are virtual regs
  - `panthor_fw_csg_output_iface` in panthor
- `CSG_ACK` see `CSG_REQ`
- `CSG_DB_ACK` see `CSG_DB_REQ`
- `CSG_IRQ_REQ` see `CSG_IRQ_ACK`
- `CSG_STATUS_EP_CURRENT`
- `CSG_STATUS_EP_REQ`
- `CSG_STATUS_STATE`
  - bit 0 (`CSG_STATUS_STATE_IS_IDLE`): CSG is idle
- `CSG_RESOURCE_DEP`

## `STREAM_CONTROL_BLOCK` regs

- these are virtual regs
  - `panthor_fw_cs_control_iface` in panthor
- `STREAM_FEATURES`
  - bit 7:0: number of regs in CSHWIF
  - bit 15:8: number of scoreboards
  - bit 16: support compute
  - bit 17: support frag
  - bit 18: support tiler
- `STREAM_INPUT_VA` va of `CS_KERNEL_INPUT_BLOCK`
- `STREAM_OUTPUT_VA` va of `CS_KERNEL_OUTPUT_BLOCK`

## `CS_KERNEL_INPUT_BLOCK` regs

- these are virtual regs
  - `panthor_fw_cs_input_iface` in panthor
- `CS_REQ`
  - `STATE` requests to start/stop a CS
  - `EXTRACT_EVENT`
  - `IDLE_SYNC_WAIT` consider the CS idle when `SYNC_WAIT` stalls
  - `IDLE_PROTM_PEND`
  - `IDLE_EMPTY` consider the CS idle when when ring buffer is empty
  - `IDLE_RESOURCE_REQ` consider the CS idle when `REQ_RESOURCE` stalls
  - `IDLE_SHARED_SB_DEC`
  - `TILER_OOM` acks `TILER_OOM`
    - gpu toggles the bit in `CS_ACK` upon tiler oom
  - `PROTM_PEND`
  - `FATAL` acks `FATAL`
    - gpu toggles the bit in `CS_ACK` upon unrecoverable err
  - `FAULT` acks `FAULT`
    - gpu toggles the bit in `CS_ACK` upon recoverable err
- `CS_CONFIG`
  - `PRIORITY` 0..15, with 15 being the highest priority within the CSG
  - `USER_DOORBELL` which doorbell to use
- `CS_ACK_IRQ_MASK`
  - irq sources to enable
  - when `CS_ACK` is updated, and the changed bits are in `CS_ACK_IRQ_MASK`,
    generate an irq
- `CS_BASE` va of the ring buffer
- `CS_SIZE` size of the ring buffer
- `CS_TILER_HEAP_START` start of extra tiler heap chunk on `TILER_OOM`
- `CS_TILER_HEAP_END` end of extra tiler heap chunk on `TILER_OOM`
- `CS_USER_INPUT` va of `CS_USER_INPUT_BLOCK`
- `CS_USER_OUTPUT` va of `CS_USER_OUTPUT_BLOCK`
- `CS_INSTR_CONFIG`
- `CS_INSTR_BUFFER_SIZE`
- `CS_INSTR_BUFFER_BASE`
- `CS_INSTR_BUFFER_OFFSET_POINTER`

## `CS_KERNEL_OUTPUT_BLOCK` regs

- these are virtual regs
  - `panthor_fw_cs_output_iface` in panthor
- `CS_ACK` see `CS_REQ`
- `CS_STATUS_CMD_PTR` reports the va of the next instr
- `CS_STATUS_WAIT` reports the wait status
  - `SB_MASK` if waiting for scoreboard entry n
  - `SB_SOURCE` is one of
    - `NONE`
    - `WAIT` if executing `WAIT` instr
  - `SYNC_WAIT_CONDITION` is `SYNC_WAIT` cond
  - `PROGRESS_WAIT` if executing `PROGRESS_WAIT` instr
  - `PROTM_PEND` if waiting for protected mode
  - `SYNC_WAIT_SIZE` is 64-bit if set; 32-bit if cleared
  - `SYNC_WAIT` if executing `SYNC_WAIT` instr
- `CS_STATUS_REQ_RESOURCE` reports requested and allocated resources
- `CS_STATUS_WAIT_SYNC_POINTER` reports va of `SYNC_WAIT`
- `CS_STATUS_WAIT_SYNC_VALUE` reports val of `SYNC_WAIT`
- `CS_STATUS_SCOREBOARDS` reports non-zero scoreboard entries
- `CS_STATUS_BLOCKED_REASON` reports the block reason
  - `UNBLOCKED` not blocked
  - `SB_WAIT` blocked by `WAIT` instr
  - `PROGRESS_WAIT` blocked by `PROGRESS_WAIT` instr
  - `SYNC_WAIT` blocked by `SYNC_WAIT` instr
  - `DEFERRED` blocked by deferred instruction
  - `RESOURCE` blocked by `REQ_RESOURCE` instr
  - `FLUSH` blocked by `FLUSH_CACHE2` instr
- `CS_STATUS_WAIT_SYNC_VALUE_HI` reports higher 32-bit of `SYNC_WAIT` val
- `CS_FAULT` recoverable error info
  - `EXCEPTION_TYPE`
    - `OK`
    - `KABOOM` shader core executes `KABOOM`
    - `CS_RESOURCE_TERMINATED` another job causes iterator to terminate
    - `CS_BUS_FAULT`
    - `CS_INHERIT_FAULT` `SYNC_WAIT` fails with an error
    - `INSTR_INVALID_PC` bad shader instr pc
    - `INSTR_INVALID_ENC` bad shader instr encoding
    - `INSTR_BARRIER_FAULT`
    - `DATA_INVALID_FAULT` invalid input data
    - `TILE_RANGE_FAULT` bad tile
    - `ADDR_RANGE_FAULT` out-of-range access
    - `IMPRECISE_FAULT` unkonwn reason
    - `RESOURCE_EVICTION_TIMEOUT`
  - `EXCEPTION_DATA`
- `CS_FATAL` unrecoverable error info
  - `EXCEPTION_TYPE`
    - `OK`
    - `CS_CONFIG_FAULT`
    - `CS_UNRECOVERABLE`
    - `CS_ENDPOINT_FAULT` no available shader cores
    - `CS_BUS_FAULT`
    - `CS_INVALID_INSTRUCTION` illegal CSF instr
    - `CS_CALL_STACK_OVERFLOW` too deep `CALL` instr
    - `FIRMWARE_INTERNAL_ERROR`
  - `EXCEPTION_DATA`
- `CS_FAULT_INFO`
- `CS_FATAL_INFO`
- `CS_HEAP_VT_START` number of vertex/tiler ops started
- `CS_HEAP_VT_END` number of vertex/tiler ops completed
- `CS_HEAP_FRAG_END` number of frag ops completed
- `CS_HEAP_ADDRESS` va of heap ctx that hits OOM

## `CS_USER_INPUT_BLOCK` regs

- these are virtual regs
  - `panthor_fw_ringbuf_input_iface` in panthor
- `CS_INSERT` is the ring buffer tail
- `CS_EXTRACT_INIT` is the initial ring buffer head, copied to `CS_EXTRACT`
  when the CS starts

## `CS_USER_OUTPUT_BLOCK` regs

- these are virtual regs
  - `panthor_fw_ringbuf_output_iface` in panthor
- `CS_EXTRACT` is the ring buffer head
- `CS_ACTIVE` set when the CS is active (scheduled on hw)

## Instructions

- sync / async / deferred
  - sync instr executes to completion before the next instr
  - async instr latches states, increments sb entry, and initiates op before
    the next instr
    - when it executes to completion, it decrements the sb entry
  - deferred instr can be sync or async
    - when async, it latches states and increments sb entry before the next instr
    - when the waited sb entries become zero, it executes to completion and
      decrements its sb entry
- native / emulated
  - a native instr is executed by CEU
  - an emulated instr is executed by MCU
- fast / slow
  - all emulated instrs are slow
  - some native instrs (control flow and sb related ones) are slow too
- alu
  - `ADD32`
  - `ADD64`
  - `ADD_IMM32`
  - `ADD_IMM64`
  - `AND32`
  - `BFEXT_S32`
  - `BFEXT_S64`
  - `BFEXT_U32`
  - `BFEXT_U64`
  - `BFINS32`
  - `BFINS64`
  - `BFINS_IMM32`
  - `BFINS_IMM64`
  - `BIT_CLEAR32`
  - `BIT_SET32`
  - `LOGIC_OP32`
  - `LSHIFT32`
  - `LSHIFT64`
  - `LSHIFT_IMM32`
  - `LSHIFT_IMM64`
  - `MOVE32`
  - `MOVE48`
  - `MOVE_REG32`
  - `NOP`
  - `NOT32`
  - `OR32`
  - `RSHIFT_IMM_S32`
  - `RSHIFT_IMM_S64`
  - `RSHIFT_IMM_U32`
  - `RSHIFT_IMM_U64`
  - `RSHIFT_S32`
  - `RSHIFT_S64`
  - `RSHIFT_U32`
  - `RSHIFT_U64`
  - `SUB32`
  - `SUB64`
  - `UMIN32`
  - `UMIN64`
  - `UMIN_IMM32`
  - `UMIN_IMM64`
  - `XOR32`
- control flows
  - `BRANCH`
  - `CALL`
  - `JUMP`
- load/store
  - `LOAD_MULTIPLE`
  - `STORE_MULTIPLE`
  - `STORE_STATE`
- other
  - `ERROR_BARRIER`
  - `FINISH_FRAGMENT`
  - `FINISH_TILING`
  - `FLUSH_CACHE2`
  - `HEAP_OPERATION`
  - `HEAP_SET`
  - `NEXT_SB_ENTRY`
  - `PROT_REGION`
  - `REQ_RESOURCE`
  - `RUN_COMPUTE`
  - `RUN_COMPUTE_INDIRECT`
  - `RUN_FRAGMENT`
  - `RUN_FULLSCREEN`
  - `RUN_IDVS2`
  - `SET_EXCEPTION_HANDLER`
  - `SET_STATE`
  - `SET_STATE_IMM32`
  - `SHARED_SB_DEC`
  - `SHARED_SB_INC`
  - `SYNC_ADD32`
  - `SYNC_ADD64`
  - `SYNC_SET32`
  - `SYNC_SET64`
  - `SYNC_WAIT32`
  - `SYNC_WAIT64`
  - `TRACE_POINT`
  - `WAIT`
