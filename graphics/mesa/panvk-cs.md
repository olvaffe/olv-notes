Mesa PanVK Command Stream
=========================

## Command Stream XML

- `genxml/v10.xml`
- CS aka CEU (Command Execution Unit)
  - 7 cs enums
    - `CS Condition`
    - `CS State`
    - `CS Heap Operation`
    - `CS Flush Mode`
    - `CS Sync scope`
    - `CS Exception type`
    - `CS Opcode`
  - 40 cs structs
    - `pandecode_cs` is a good place to know how they work
      - `interpret_ceu_instr`
      - `disassemble_ceu_instr`
    - all structs have size 2 (dwords)
    - `CS Base` is not a real opcode
      - it is only used by the decoder
      - bit 0..55: op payload
      - bit 56..63: op code
    - `CS NOP` is nop
      - the payload is ignored
    - `CS MOVE` does `dst_reg = imm48`
    - `CS MOVE32` does `dst_reg = imm32`
    - `CS WAIT` waits the specified slots (scoreboards?)
      - the fw supports `scoreboard_slot_count` slots
      - userspace can decide which is for what
      - `PANVK_SB_LS`, load/store
      - `PANVK_SB_IMM_FLUSH`,
      - `PANVK_SB_DEFERRED_SYNC`
      - `PANVK_SB_DEFERRED_FLUSH`
      - `PANVK_SB_ITER_START`
      - `PANVK_SB_ITER_COUNT`
    - `CS RUN_COMPUTE` runs a compute job
      - it expects reg0..reg39 to be set up, such as
        - reg0 points to res table
        - reg8 points to push consts
        - reg16 points to shader
        - reg24 points to local storage
        - reg32 is global attr offset
        - reg33 is wg size
        - reg34..36 are wg offsets
        - reg37..39 are wg counts
    - `CS RUN_TILING` runs a tiling job, unused?
    - `CS RUN_IDVS` runs an idvs (index-driven vs) job
      - traditional vs: vertex shading -> primitive assembly -> culling
        - wasted computation if a primitive is culled
      - idvs since bifrost: primitive assembly -> position shading -> culling -> varying shading
    - `CS RUN_FRAGMENT` runs an fs job
    - `CS RUN_FULLSCREEN` runs a fullscreen job
    - `CS FINISH_TILING`
    - `CS FINISH_FRAGMENT`
    - `CS ADD_IMMEDIATE32` does `dst_reg = src_reg + imm32`
    - `CS ADD_IMMEDIATE64` does `dst_reg64 = src_reg64 + imm32`
    - `CS UMIN32` does `dst_reg = min(src_reg1, src_reg2)`
    - `CS LOAD_MULTIPLE` does `dst_reg = load(src_reg + imm16)`
    - `CS STORE_MULTIPLE` does `store(src_reg1 + imm16, src_reg2)`
    - `CS BRANCH` jumps a signed imm16 offset if the reg meets the condition
    - `CS SET_SB_ENTRY` selects the scoreboard slots for endpoint tasks (frag
      & comp) and other tasks (tiler)
    - `CS PROGRESS_WAIT` is unused
    - `CS SET_EXCEPTION_HANDLER` is unused
    - `CS CALL` calls `(reg1, reg2)`
      - `reg1` is the addr
      - `reg2` is the length
    - `CS JUMP` jumps to `(reg1, reg2)`
      - why does it need the length?
    - `CS REQ_RESOURCE` is needed before/after running a job
      - it requests and releases the needed res?
    - `CS FLUSH_CACHE2` flushes L2 and LSC caches
      - this is used for barriers
    - `CS SYNC_ADD32`  does `store(addr, load(addr) + src_reg)` after sync
    - `CS SYNC_SET32` does `store(addr, src_reg)` after sync
    - `CS SYNC_WAIT32` busy-waits until an `load(addr)` meets the condition
    - `CS STORE_STATE` does `store(src_reg + imm16, state)`, where `state` is
      - `MALI_CS_STATE_TIMESTAMP`
      - `MALI_CS_STATE_CYCLE_COUNT`
      - `MALI_CS_STATE_DISJOINT_COUNT`
      - `MALI_CS_STATE_ERROR_STATUS`
    - `CS PROT_REGION` is unused
    - `CS PROGRESS_STORE` is unused
    - `CS PROGRESS_LOAD` is unused
    - `CS RUN_COMPUTE_INDIRECT` runs a compute job
    - `CS ERROR_BARRIER` is unused
    - `CS HEAP_SET` sets the heap va
    - `CS HEAP_OPERATION`
    - `CS TRACE_POINT` is unused
    - `CS SYNC_ADD64`
    - `CS SYNC_SET64`
    - `CS SYNC_WAIT64`

## panloader

- no support for csf
- build
  - clone <https://gitlab.freedesktop.org/panfrost/panloader.git> to
    `$mesa/src/panfrost/lib/panloader`
  - `meson setup out`
    - it expects panfrost has been built under `$mesa/build`
- `__attribute__((constructor))`
  - `panloader_constructor` inits a recursive mutex
  - `constructor_util` calls `pandecode_create_context` to init
    `pandecode_ctx`
- macros
  - `PROLOG` looks up and saves the original symbol address
  - `LOCK` and `UNLOCK` locks/unlocks the recursive mutex
- `open` and `open64`
  - if the path is `/dev/mali0`, save the fd to `mali_fd`
- `close`
  - reset `mali_fd` if it is closed
- `ioctl`
  - `dvalin` is defined
  - `MALI_IOCTL_GET_GPUPROPS` is parsed to set `product_id` and `bifrost`
  - `MALI_IOCTL_MEM_ALLOC` is `panwrap_track_allocation`ed
    - the alloc is tracked in `allocations` and is `pandecode_inject_mmap`ed
  - `MALI_IOCTL_JOB_SUBMIT` is `pandecode_jc`ed
- `mmap` and `mmap64`
  - if the mmap is against `mali_fd`, `offset` is gpu va
    - iirc, gpu va also uniquely identifies a bo and is used as the handle
  - `MEM_MAP_TRACKING_HANDLE` is a special offset
  - otherwise, `panwrap_track_mmap` calls `pandecode_inject_mmap` to track the
    mapping

## pandecode

- usage
  - `pandecode_create_context` and `pandecode_destroy_context` are called
    per-VkDevice
  - `pandecode_inject_mmap` and `pandecode_inject_free` are called per-BO
    - `ctx->mmap_tree` manages the BOs
  - these are called per-queue-submit
    - `pandecode_dump_file_open` opens the file
      - optional because `pandecode_cs` and `pandecode_dump_mappings` (but not
        `pandecode_log`) open on-demand
    - `pandecode_log` printfs to the file
    - `pandecode_cs` decodes a command stream
    - `pandecode_dump_mappings` dumps the contents of BOs
    - `pandecode_next_frame` closes the file
- `pandecode_cs` calls `GENX(pandecode_cs)`
  - `regs` is an array of 256 u32, initialized to 0, on stack
  - `pandecode_find_mapped_gpu_mem_containing` finds the BO containing the cs
    - it also `mprotect` the BO and adds it to `ctx->ro_mappings`
    - at the end of decode, `pandecode_map_read_write` undoes the `mprotect`
  - `disassemble_ceu_instr` prints the u64 instr
  - `interpret_ceu_instr` simulates the u64 instr
- `pandecode_run_idvs` decodes `RUN_IDVS`
  - `d0`, `d2`, and `d4` are SRT
    - vs may be separated into position shader and varying shader
    - position shader always uses `d0`
    - varying shader uses `d0` or `d2`, depending on a bit of the instr
    - fragment shader uses `d0` or `d4`, depending on a bit of the instr
    - `GENX(pandecode_resource_tables)` decodes SRT
  - `d8`, `d10`, and `d12` are FAU
    - position shader always uses `d8`
    - varying shader uses `d8` or `d10`, depending on a bit of the instr
    - fragment shader always uses `d12`
    - `GENX(pandecode_fau)` decodes FAU
  - `d16`, `d18`, and `d20` are SPD (shader program descriptor?)
    - position shader always uses `d16`
    - varying shader always uses `d18`, if separated from position shader
    - fragment shader always uses `d20`
    - `GENX(pandecode_shader)` decodes SPD
  - `d24`, `d26`, and `d28` are TSD
    - position shader always uses `d24`
    - varying shader uses `d24` or `d26`, depending on a bit of the instr
    - fragment shader uses `d24` or `d28`, depending on a bit of the instr
  - `r32` is `Global attribute offset`
  - `r33` is `Index count`
  - `r34` is `Instance count`
  - `r35` is `Index offset`
  - `r36` is `Vertex offset`
  - `r37` is `Instance offset`
  - `r38` is `Tiler DCD flags2`
  - `r39` is `Index array size`
  - `d40` is tiler context
    - `GENX(pandecode_tiler)` decodes tiler context
  - `d42` is `Scissor`
  - `r44` is `Low depth clamp`
  - `r45` is `High depth clamp`
  - `d46` is `Occlusion`
  - `r48` is `Varying allocation`
  - `d50` is `Blend`
    - `GENX(pandecode_blend_descs)` decodes blend
  - `d52` is `Depth/stencil`
  - `d54` is `Indices`
  - `r56` is `Primitive flags`
  - `r57` is `DCD Flags 0`
  - `r58` is `DCD Flags 1`
  - `r60` is `Primitive size`
- `pandecode_run_fragment` decodes `RUN_FRAGMENT`
  - `d40` is FBD (framebuffer descriptor?)
    - `GENX(pandecode_fbd)` decodes FBD
  - `d42` is `Scissor`
- `pandecode_run_compute` decodes `RUN_COMPUTE`
  - `d0`, `d2`, `d4`, `d6` are SRT (resource table)
    - the instr selects one of them
    - `GENX(pandecode_resource_tables)` decodes SRC
  - `d8`, `d10`, `d12`, `d14` are FAU (fast access uniform, aka push consts)
    - the instr selects one of them
    - `GENX(pandecode_fau)` decodes FAU
  - `d16`, `d18`, `d20`, `d22` are SPD (shader)
    - the instr selects one of them
    - `GENX(pandecode_shader)` decodes SPD
  - `d24`, `d26`, `d28`, `d30` are TSD (local storage)
  - `r32` is `Global attribute offset`
  - `r33` is `Workgroup size`
  - `r34` is `Job offset X`
  - `r35` is `Job offset Y`
  - `r36` is `Job offset Z`
  - `r37` is `Job size X`
  - `r38` is `Job size Y`
  - `r39` is `Job size Z`

## CS Builder

- `cs_builder`
  - `cs_builder_conf`
    - `nr_registers` is the number of regs and is arch-dependent
    - `nr_kernel_registers` is the number of regs at the top that the kmd uses
      - `cs_builder` itself also uses the top 3 regs for chunk linking
    - `alloc_buffer` allocates a new bo for instrs
    - `ls_tracker` validates loads/stores and is for debugging
    - `reg_perm` validates reg accesses and is for debugging
  - a `cs_chunk` is a BO
    - multiple chunks are linked together by `JUMP` instr
    - `root_chunk` is the first BO
    - `cur_chunk` is the current BO
  - `blocks` is for control flow
    - `stack` is the current cfg node, if non-null
    - `instrs` is a temporary storage for instrs belonging to a control flow
      - this is done to avoid chunk linking in the middle of a control flow
    - `pending_if`
  - `length_patch` is for chunk linking
    - `JUMP` requires a va and a length, but the length of the next chunk is
      unkonwn yet
    - `length_patch` points to the length field, which will be patched in when
      the size of the next chunk is known
  - `discard_instr_slot` is used only when bo allocation fails
- `cs_alloc_ins_block` returns a pointer to a u64 for the next instr(s)
  - if `b->blocks.stack` is non-NULL, we are in a control flow and instrs are
    stashed to `b->blocks.instrs` temporarily
    - `cs_flush_block_instrs` will copy them to the current chunk
  - if `b->cur_chunk` is full,
    - allocates a new bo
    - sets up a jump to the new bo from the current bo
      - `cs_overflow_address_reg` and `cs_overflow_length_reg` use the top 3
        regs
    - `cs_wrap_chunk` ends the current chunk
    - `b->length_patch` is set up (which will be patched to the len of the new
      chunk)
    - `b->cur_chunk` is updated
  - increments `b->cur_chunk` and returns a ptr in `b->cur_chunk`
- `cs_scratch_reg32` returns a `cs_index`
  - it checks that the reg is in `PANVK_CS_REG_SCRATCH_{START,END}`
  - `cs_reg_tuple` builds the `cs_index` struct
  - there is a total of `b->conf.nr_registers - b->conf.nr_kernel_registers`
    registers
- `cs_move32_to`
  - `cs_alloc_ins` returns a ptr to an instr
    - the ptr points to an offset in `b->cur_chunk.buffer.cpu` which is u64
  - `pan_pack` expands to
    - `struct MALI_CS_MOVE32 I = { MALI_CS_MOVE32_header }`
    - custom code to modify `I`
    - `MALI_CS_MOVE32_pack(ptr, &I)`
  - `cs_dst32` converts a `cs_index` into a u8, the raw reg number
- `cs_branch` skips offset if `cond(val)` is true
- control flow
  - `cs_block_start` and `cs_block_end` start/end a cfg node
    - they update `b->blocks.stack`
  - `cs_while(MALI_CS_CONDITION_ALWAYS)`
    - `cs_while_start`
      - `cs_block_start` adds a block
      - `cs_set_label(b, &loop->start)` updates `loop->start` to target start
        of block
    - `cs_while_end`
      - `cs_branch_label(b, &loop->start, ...)` emits `BRANCH` instr to jump
        back to the start of block if `cond(val)` is true
      - `cs_set_label(b, &loop->end)` updates `loop->end` to target end of
        block
        - if `cs_break` was used, it also patches all `BREAK` instrs
      - `cs_block_end` ends the block
        - `cs_flush_block_instrs` copies stashed cfg node instrs into the
          current chunk
    - if there is `cs_continue`, `cs_loop_conditional_continue` calls
      `cs_branch_label(b, &loop->start, ...)` to `BRANCH` backward to the start
    - if there is `cs_break`, `cs_loop_conditional_break` calls
      `cs_branch_label(b, &loop->end, ...)` to `BRANCH` forward to the end
      - because the location of the end is unknown, we save the location of
        the `BRANCH` in `last_forward_ref` for later patching
      - if there are multiple breaks, we use the `offset` field of `BRANCH`
        instrs to form a list
  - `cs_match`
    - `cs_match_start`
      - `cs_block_start`
    - `cs_case(N)`
      - if there is a prior `cs_case(M)`
        - it calls `cs_branch_label(b, &match->break_label, ...)` to jump to
          the end for the prior case
          - that is, if the prior case is a match, it jumps to the end
        - it calls `cs_set_label(b, &match->next_case_label)` to point
          `next_case_label` to the start for the current case
          - that is, if the prior case is not a match, it jumps to this case
        - it reinitializes `next_case_label`
      - it calls `cs_branch_label(b, &match->next_case_label, ...)` with the
        condition `(val - N) != 0`
        - that is, it jumps to the next case if `val` is not N
    - `cs_default`
      - it calls `cs_branch_label(b, &match->break_label, ...)` to jump to the
        end for the prior case
        - that is, if the prior case is a match, it jumps to the end
      - it calls `cs_set_label(b, &match->next_case_label)`
        - that is, if the prior case is not a match, it jumps to the this
          defualt case
      - it reinitializes `next_case_label`
    - `cs_match_end`
      - `cs_set_label` is called on both `next_case_label` and `break_label`
      - `cs_block_end`
  - `cs_if`
    - `cs_if_start`
      - `cs_block_start` adds a block
      - `cs_branch_label` branches forward to the end if `!cond(val)`
    - `cs_if_end`
      - rather than calling `cs_block_end`, it saves the block to
        `b->blocks.pending_if`
      - later, `cs_flush_pending_if` will update `b->blocks.stack`
      - this is done such that `cs_else` has a chance to patch `cs_if`?
- panvk-specific helpers
  - `panvk_cs_reg_whitelist` is a macro to define a `reg_perm_cb_t`
    - it validates that the regs being written are on the whitelist
  - `cs_update_progress_seqno(b) { foo }` validates that foo only writes to
    seqno regs (84..89)
    - 2 regs per subqueue
    - `PANVK_SUBQUEUE_VERTEX_TILER` uses 84 and 85
    - `PANVK_SUBQUEUE_FRAGMENT` uses 86 and 87
    - `PANVK_SUBQUEUE_COMPUTE` uses 88 and 89
  - `cs_update_compute_ctx(b) { foo }` validates that foo only writes to
    compute regs (0..39)
  - `cs_update_frag_ctx(b) { foo }` validates that foo only writes to
    frag regs (40..46)
  - `cs_update_vt_ctx(b) { foo }` validates that foo only writes to
    vertex tiler / idvs regs (0..60)

## Command Pool

- a `panvk_cmd_pool` is a subclass of `vk_command_pool`
  - `cs_bo_pool`
  - `desc_bo_pool`
  - `varying_bo_pool`
  - `tls_bo_pool`
  - `push_sets`
- `panvk_per_arch(cmd_buffer_ops)`
  - `panvk_create_cmdbuf` creates a cmdbuf
  - `panvk_reset_cmdbuf` resets a cmdbuf
  - `panvk_destroy_cmdbuf` destroys a cmdbuf
- a `panvk_cmd_buffer` is a subclass of `vk_command_pool`
  - `cs_pool` uses `pool->cs_bo_pool` as storage
  - `desc_pool` uses `pool->desc_bo_pool` as storage
  - `tls_pool` uses `pool->tls_bo_pool` as storage
  - `push_sets`
  - `state`
    - `gfx`
    - `compute`
    - `push_constants`
    - `cs`
    - `tls`

## Command Buffer

- a `panvk_cmd_buffer` has
  - a `panvk_cmd_graphics_state`
  - a `panvk_cmd_compute_state`
  - a `panvk_push_constant_state`
    - this is user push consts
    - sysvals are also pushed on top of user push consts
  - a per-subqueue `panvk_cs_state`
  - a `panvk_tls_state`
    - tls is for shader reg spills, etc.
- subqueue progress
  - `init_subqueue` inits `syncobjs[subqueue].seqno` to 1
  - when an action op (dispatch, draw, etc.) is emitted to a cmdbuf
    - there is a `cs_sync64_add` to increments the seqno after the action op
      completes
    - `cmdbuf->state.cs[subqueue].relative_sync_point` is also incremented
  - after a cmdbuf is recorded, `finish_cs` increments
    `cs_progress_seqno_reg()` by `relative_sync_point` for each subqueue
  - when `panvk_per_arch(CmdPipelineBarrier2)` emits a barrier,
    - there is a `cs_sync64_wait` to wait until
      `syncobjs[subqueue].seqno > cs_progress_seqno_reg() + relative_sync_point`
  - remember that cmdbuf recording and submission are separated
    - `syncobjs[subqueue].seqno` is the number of completed ops on the subqueue
      since initialization
    - `cs_progress_seqno_reg()` is the number of completed ops on the subqueue
      when the cmdbuf starts executing
    - `relative_sync_point` is the number of ops recorded in the cmdbuf so far
- compute pipeline
  - `panvk_per_arch(BeginCommandBuffer)`
  - `panvk_per_arch(CmdDispatchBase)`
    - `GENX(pan_emit_tls)`
    - emits to `PANVK_SUBQUEUE_COMPUTE`
    - `panvk_per_arch(cs_pick_iter_sb)`
    - `cs_run_compute`
    - update seqno
  - `panvk_per_arch(EndCommandBuffer)`
    - `emit_tls` emits to `cmdbuf->state.tls.desc.cpu`
    - `finish_cs` on all subqueues
      - it accumulates `relative_sync_point` to `cs_progress_seqno_reg()`
- graphics pipeline
  - `panvk_per_arch(BeginCommandBuffer)`
  - `panvk_per_arch(CmdBeginRendering)`
  - `panvk_cmd_draw`
    - `update_tls`
    - `get_tiler_desc` inits `cmdbuf->state.gfx.render.tiler`
    - `get_fb_descs` inits `cmdbuf->state.gfx.render.fbds`
    - emits to `PANVK_SUBQUEUE_VERTEX_TILER`
  - `panvk_per_arch(CmdEndRendering)`
    - if there is no draw, `get_tiler_desc` and `get_fb_descs` haven't been
      called yet
      - if there is clear, `get_fb_descs` is called now
    - `flush_tiling`
      - `cs_finish_tiling` flushes tiling ops
      - `LOAD scrach01, [subq->ctx + syncobjs]` to load syncobj
      - `LOAD scrach2, [subq->ctx + iter_sb]`
      - `cs_heap_operation`
      - `cs_sync64_add` to increment syncobj
      - `STORE scratch2 [subq->ctx + syncobjs]` to store syncobj
    - `issue_fragment_jobs`
      - `wait_finish_tiling`
        - `cs_sync64_wait` until `[scratch01]` is greater than
          `progress_reg + relative_sync_point`
    - `resolve_attachments`
  - `panvk_per_arch(EndCommandBuffer)`
- `panvk_per_arch(CmdPipelineBarrier2)`
  - `panvk_per_arch(cmd_flush_draws)`
    - `flush_tiling`
    - `issue_fragment_jobs`
    - `force_fb_preload`
  - it looks at `wait_sb_mask` for each subqueue
    - `cs_wait_slots` to wait for scoreboard slots
    - if cache flush, `cs_flush_caches` and `cs_wait_slot`
    - if another subqueue waits on this subqueue, `cs_sync64_add`
  - it looks at `wait_subqueue_mask` for each subqueue, and if subqueue j is
    on `wait_subqueue_mask` of subqueue i,
    - `cs_sync64_wait` such that subqueue i waits for subqueue j
- scoreboard slots
  - slot 0 is for `PANVK_SB_LS` and `PANVK_SB_IMM_FLUSH`
  - slot 1 is for `PANVK_SB_DEFERRED_SYNC`
  - slot 2 is for `PANVK_SB_DEFERRED_FLUSH`
  - slot 3..7 are for iterators
    - each subqueue initializes `iter_sb` to 0
    - `panvk_per_arch(cs_pick_iter_sb)` picks `iter_sb`
    - `cs_match(b, iter_sb, cmp_scratch)` increments `iter_sb`
  - `panvk_per_arch(cs_pick_iter_sb)`
    - `LOAD scratch0, [subctx->iter_sb]`
    - `WAIT scratch0`
    - `SET_SB_ENTRY scratch0, LS`

## Command Buffer Example

- assume this call sequence to clear a color image
  - `vkBeginCommandBuffer` to begin cmd
  - `vkCmdPipelineBarrier` to transition an image from undef to xfer dst
    - src: `VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`, no access
    - dst: `VK_PIPELINE_STAGE_TRANSFER_BIT`, `VK_ACCESS_TRANSFER_WRITE_BIT`
  - `vkCmdClearColorImage` to clear the image
  - `vkCmdPipelineBarrier` to transition the image to general
    - src: `VK_PIPELINE_STAGE_TRANSFER_BIT`, `VK_ACCESS_TRANSFER_WRITE_BIT`
    - dst: `VK_PIPELINE_STAGE_HOST_BIT`, `VK_ACCESS_HOST_READ_BIT`
  - `vkEndCommandBuffer` to end cmd
- first `panvk_per_arch(CmdPipelineBarrier2)` emits no instrs to any of the
  subqueue
- `panvk_per_arch(CmdClearColorImage)` emits instrs to
  `PANVK_SUBQUEUE_VERTEX_TILER`
  - `panvk_per_arch(cmd_meta_gfx_start)`
  - `vk_meta_clear_color_image`
    - `vk_meta_create_image_view`
    - `panvk_per_arch(CmdBeginRendering)`
    - `vk_meta_clear_attachments`
      - `vk_meta_create_pipeline_layout`
      - `vk_meta_create_graphics_pipeline`
      - `vk_common_CmdBindPipeline` binds the clear pipeline
      - `panvk_per_arch(CmdPushConstants2KHR)` pushes the clear value
      - `vk_meta_draw_rects`
        - `vk_common_CmdSetViewport`
        - `vk_common_CmdSetScissor`
        - `vk_meta_create_buffer`
        - `panvk_meta_cmd_bind_map_buffer`
        - `panvk_per_arch(CmdDraw)`
    - `panvk_per_arch(CmdEndRendering)`
  - `panvk_per_arch(cmd_meta_gfx_end)`
- looking at `panvk_per_arch(CmdDraw)` alone,
  - `update_tls`
    - allocs `MALI_LOCAL_STORAGE` and writes va to `d24`
  - `get_tiler_desc`
    - allocs `MALI_TILER_CONTEXT`, inits it, and writes va to `d40`
    - loads `tiler_heap` and `geom_buf` from subq ctx to scratches and stores
      them to `MALI_TILER_CONTEXT`
    - zeros bottom half of `MALI_TILER_CONTEXT`
    - `panvk_per_arch(cs_pick_iter_sb)` to pick the iter sb
    - `cs_heap_operation(MALI_CS_HEAP_OPERATION_VERTEX_TILER_STARTED)`
  - `get_fb_descs` allocs fb descs
    - `MALI_FRAMEBUFFER`
    - `MALI_ZS_CRC_EXTENSION`
    - one or more `MALI_RENDER_TARGET`
  - `panvk_per_arch(cmd_prepare_push_descs)`
  - `prepare_sysvals`
  - `prepare_push_uniforms`
    - `panvk_per_arch(cmd_prepare_push_uniforms)` allocs and inits push consts
      - first half is user push consts
      - second half is sysvals
    - push const va is written to `d8` and `d12`
      - push const is called FAU, fast access uniform
  - `prepare_vs`
    - `prepare_vs_driver_set` allocs and inits vs driver set
      - `MALI_ATTRIBUTE` for vs attrs
      - dummy `MALI_SAMPLER`
      - `panvk_per_arch(cmd_fill_dyn_bufs)` for `MALI_BUFFER`
      - more `MALI_BUFFER` for vbs
    - `panvk_per_arch(cmd_prepare_shader_res_table)`
      - `MALI_RESOURCE` array for the driver set and the user desc sets
    - writes res table va to `d0`
    - writes pos shader va to `d16`
    - no varying shader and no write to `d18`
  - `prepare_fs`
    - `prepare_fs_driver_set` allocs and inits vs driver set
      - dummy `MALI_SAMPLER`
      - `panvk_per_arch(cmd_fill_dyn_bufs)` for `MALI_BUFFER`
    - `panvk_per_arch(cmd_prepare_shader_res_table)`
      - `MALI_RESOURCE` array for the driver set and the user desc sets
    - writes res table va to `d4`
    - writes frag shader va to `d20`
  - writes draw info to `r32` to `r39`
    - vertex base, vertex count, instance count, index offset, etc.
  - no ib and no write to `d54`
  - writes to `r56` and `r48`
  - `prepare_blend`
    - allocs and inits `MALI_BLEND`
    - `panvk_per_arch(blend_emit_descs)`
    - writes blend va to `d50`
  - `prepare_ds`
    - allocs and inits `MALI_DEPTH_STENCIL`
    - writes ds va to `d52`
  - `prepare_dcd`
    - writes `MALI_DCD_FLAGS_0` to `r57`
      - kill pixel, front face ccw, cull front/back, msaa, etc.
    - writes `MALI_DCD_FLAGS_1` to `r58`
      - sample mask, color write mask, etc.
  - `prepare_vp`
    - writes `MALI_SCISSOR` to `d42`
    - writes viewport min/max depth to `r44` and `r45`
  - `cs_req_res(CS_IDVS_RES)`
  - `cs_run_idvs`
  - `cs_req_res(0)`
- looking at `panvk_per_arch(CmdEndRendering)` alone,
  - `flush_tiling` emits to `PANVK_SUBQUEUE_VERTEX_TILER`
    - `cs_req_res(CS_TILER_RES)`
    - `cs_finish_tiling`
    - `cs_req_res(0)`
    - `cs_heap_operation(MALI_CS_HEAP_OPERATION_VERTEX_TILER_COMPLETED)`
    - `cs_sync64_add` to increment seqno
    - increment `subq->iter_sb`
  - `issue_fragment_jobs` emits to `PANVK_SUBQUEUE_FRAGMENT`
    - `wait_finish_tiling`
      - `cs_sync64_wait` for `cs_sync64_add` in `flush_tiling`
        - that is, wait for `flush_tiling`
    - `panvk_per_arch(cs_pick_iter_sb)` to pick the iter sb
    - writes fb size to `r42` to `r43`
    - `prepare_fb_desc`
      - `GENX(pan_preload_fb)`
      - `GENX(pan_emit_fbd)`
        - `MALI_FRAMEBUFFER`
        - `MALI_ZS_CRC_EXTENSION`
        - `MALI_ZS_CRC_EXTENSION`
    - writes fbd va to `d48`
    - writes tiler ctx va to `d50`
    - writes layer count to `r47`
    - for each layer
      - patches tiler ctx va into fbd
      - writes fbd va and fbd flags to `d40`
      - `cs_req_res(CS_FRAG_RES)`
      - `cs_run_fragment`
      - `cs_req_res(0)`
      - increments fbd va
      - decrements layout count
    - `cs_finish_fragment`
    - `cs_sync64_add` to increment seqno
  - `resolve_attachments`
- second `panvk_per_arch(CmdPipelineBarrier2)` emits instrs to all subqueues
  - `collect_cs_deps`
    - because `VK_PIPELINE_STAGE_TRANSFER_BIT` is possible on
      `PANVK_SUBQUEUE_FRAGMENT` and `PANVK_SUBQUEUE_COMPUTE`
      - `deps->src[i].wait_sb_mask` has all iterator slots on both of them
      - `deps->src[i].cache_flush` has `MALI_CS_FLUSH_MODE_CLEAN` and
        `MALI_CS_FLUSH_MODE_CLEAN` on both of them
    - no `deps->dst[i].wait_subqueue_mask`
  - only `cs_wait_slots` and `cs_flush_caches` on the relevant subqueues
  - no `cs_sync64_wait` (for cross-subqueue sync) necessary
- `panvk_per_arch(EndCommandBuffer)` emits instrs to all subqueues
  - `finish_cs`
    - if there is any op in a subqueue, `cs_progress_seqno_reg` is updated
    - all scoreboard slots are waited and all caches are flushed
- how about `panvk_per_arch(CmdDispatchBase)`?
  - allocs `MALI_LOCAL_STORAGE`
  - `GENX(pan_emit_tls)` inits `MALI_LOCAL_STORAGE`
  - `panvk_per_arch(cmd_prepare_push_descs)`
  - `prepare_driver_set`
  - `prepare_push_uniforms`
  - `panvk_per_arch(cmd_prepare_shader_res_table)`
  - `cs_update_compute_ctx`
    - writes res table va to `d0`
    - writes push const (fau) va to `d8`
    - writes shader va to `d16`
    - writes tsd va to `d24`
    - writes 0 to `r32`
    - writes wg info to `r33` to `r39`
      - wg size, offsets, counts
  - `panvk_per_arch(cs_pick_iter_sb)`
  - `cs_req_res(CS_COMPUTE_RES)`
  - `cs_run_compute`
  - `cs_req_res(0)`
  - `cs_sync64_add` to increment seqno
  - increment `subq->iter_sb`

## Tiler

- binning
  - umd compiles vs into two shaders
    - a pos shader, which is vs stripped down to output only `gl_Position`
    - a varying shader, which is vs stripped down to output all attrs but
      `gl_Position`
  - hw runs the pos shader, assembles/culls the primitives, and the tiler
    records which triangles touches which 16x16 tiles
    - in practice, it is more complex because of hierarchical tiling
      - depending on the primitive size, the hw may bin it to 16x16, 32x32,
        64x64, and all the way to 4096x4096 tiles
    - the recorded result is called a polygon list
      - polygon list memory on midgard
        - a 64-byte prologue, if hierarchical
          - umd controls whether hierarchical tiling is enabled
        - each tile needs a 8-byte header and a 512-byte body
          - umd knows the number of tiles at each hierarchical level
      - on valhall, it uses the tiler heap?
  - hw runs the varying shader and caches the results in the tiler heap
    - it is growable and managed by kmd
    - when the hw runs out of heap, it generates an irq and the kmd grows the
      heap
- `init_tiler`
  - `queue->tiler_heap.desc` ia a `4KB + 64KB` bo
    - the first 4KB is `MALI_TILER_HEAP` descriptor
    - the rest 64KB is for geometry buffer (polygon list?)
  - `queue->tiler_heap.context` is a kernel-managed growable tiler heap
    - `tiler_heap_ctx_gpu_va` is the va of a 32-byte heap context
    - `first_heap_chunk_gpu_va` is the va of the first heap chunk
      - the heap is a list of BOs linked together, making it growable
      - the first u64 of each BO is the va of the next BO in the list
    - the fw generates an irq when it runs out of the tiler heap
      - this gives the kmd a chance to allocate more chunks
    - `chunk_size`, `initial_chunk_count`, and `max_chunks` configure the heap
      chunks
      - kmd will stop allocating new chunks when it reaches `max_chunks`,
        causing the fw to stall
    - `target_in_flight` is the max number of in-flight draws
      - each draw goes through tiling and fragment shading
      - this tells the kmd to stop allocating new chunks when too many draws
        have started but have not completed
- `init_subqueue`
  - the vas of `MALI_TILER_HEAP` and the 64KB geometry buffer are saved to
    `subq->context`
    - `MALI_TILER_HEAP` describes the tiler heap managed by kmd
  - `cs_heap_set` sets the heap ctx va
    - this is the 32-byte heap ctx allocated by kmd
  - `PANVK_SUBQUEUE_VERTEX_TILER` and `PANVK_SUBQUEUE_FRAGMENT` share the same
    `MALI_TILER_HEAP`, geometry buffer, and the heap ctx
- `panvk_cmd_draw`
  - `get_tiler_desc` inits the tiler context once per-pass
    - it allocs and inits a `MALI_TILER_CONTEXT`
      - `heap`, `geometry_buffer_size`, and `geometry_buffer` are loaded from
        `subq->context`, but why?
    - it writes the tiler ctx va to `d40`
    - `cs_heap_operation(MALI_CS_HEAP_OPERATION_VERTEX_TILER_STARTED)`
  - `cs_run_idvs` runs the job per-draw
- `panvk_per_arch(CmdEndRendering)`
  - `flush_tiling`
    - `cs_finish_tiling` waits for `cs_run_idvs`
    - `cs_heap_operation(MALI_CS_HEAP_OPERATION_VERTEX_TILER_COMPLETED)`
  - `wait_finish_tiling` waits for `PANVK_SUBQUEUE_VERTEX_TILER`
    - all prior cmds are emitted into `PANVK_SUBQUEUE_VERTEX_TILER`
    - this wait and all following cmds will be emitted into
      `PANVK_SUBQUEUE_FRAGMENT`
  - `issue_fragment_jobs`
    - assuming we only render to a single layer
    - it stores the va of `MALI_TILER_CONTEXT` to
      `MALI_FRAMEBUFFER_PARAMETERS` at offset 56, which is the `tiler` field
    - it writes the va of `MALI_FRAMEBUFFER_PARAMETERS` to `d40`
    - `cs_run_fragment` runs the fs
    - it loads from `MALI_TILER_CONTEXT` at offset 40, which are the
      `completed_top` and `completed_bottom` fields
      - these mark the range of heap chunks used by this render pass?
    - `cs_finish_fragment` waits for `cs_run_fragment`
      - it takes `completed_top` and `completed_bottom`, but for what?

## Tile Size, Load, Store, Blend

- `panvk_per_arch(CmdBeginRendering)`
  - `panvk_cmd_begin_rendering_init_state` handles load/store ops
    - `VK_ATTACHMENT_LOAD_OP_LOAD` becomes `preload`
    - `VK_ATTACHMENT_LOAD_OP_CLEAR` becomes `clear`
    - it ignores store ops and assumes `VK_ATTACHMENT_STORE_OP_STORE`
- `panvk_per_arch(CmdEndRendering)`
  - `issue_fragment_jobs` calls `prepare_fb_desc`
    - `GENX(pan_preload_fb)`
      - it allocs and inits pre-frame dcds, which are an array of 3
        `MALI_DRAW`
      - each `MALI_DRAW` runs a dcd pipeline, which is like a fs
      - `fb->bifrost.pre_post.modes[0]` is for rt preload and is
        `MALI_PRE_POST_FRAME_SHADER_MODE_ALWAYS`
      - `fb->bifrost.pre_post.modes[1]` is for zs preload and is
        `MALI_PRE_POST_FRAME_SHADER_MODE_EARLY_ZS_ALWAYS`
    - `GENX(pan_emit_fbd)`
      - it picks the tile size
        - 16x16 or smaller
      - `MALI_FRAMEBUFFER`
        - `cfg.pre_frame_0` is for rt preload
        - `cfg.pre_frame_1` is for zs preload
        - `cfg.post_frame` is disabled
        - `cfg.frame_shader_dcds` is the dcd shaders
      - `MALI_ZS_CRC_EXTENSION`
        - `ext->{zs,s}_writeback_base` is the writeback (store) offset
        - `ext->{zs,s}_writeback_row_stride` is the row stride
        - `ext->{zs,s}_writeback_surface_stride` is the surf stride
        - `ext->{zs,s}_write_format` is the format
        - `ext->{zs,s}_block_format` is the tiling
      - `MALI_RENDER_TARGET`
        - `cfg->rgb.base` is the writeback (store) offset
        - `cfg->rgb.row_stride` is the row stride
        - `cfg->rgb.surface_stride` is the surf stride
        - `cfg->writeback_format` is the format
        - `cfg->writeback_block_format` is the tiling
- `panvk_cmd_draw` calls `prepare_blend`
  - it allocs an array of `MALI_BLEND`
  - `panvk_per_arch(blend_emit_descs)` inits the array
    - `blend_needs_shader` determines if an rt needs a blend shader
      - a blend shader is needed when there is no fixed-func support, such as
        logicop, not-blendable formats, etc.
    - `GENX(pan_blend_create_shader)` creates the nir shader
    - `cfg.internal.mode` is the blend mode
      - `MALI_BLEND_MODE_OFF`, no rt or no color mask
      - `MALI_BLEND_MODE_OPAQUE`, opaque color (no blending)
      - `MALI_BLEND_MODE_FIXED_FUNCTION`, fixed-func blending
      - `MALI_BLEND_MODE_SHADER`, blend shader
  - the array va and size are written to `d50`

## Common Shader States

- all `RUN_*` instrs use 4 common shader states
  - SRT, specified in `d0`, `d2`, `d4`, `d6`
  - FAU, specified in `d8`, `d10`, `d12`, `d14`
  - SPD, specified in `d16`, `d18`, `d20`, `d22`
  - TSD, specified in `d24`, `d26`, `d28`, `d30`
- an SRT, shader resource table, is an array of `MALI_RESOURCE`
  - each `MALI_RESOURCE` corresponds to a descriptor set
    - it has an `address` field that points to an array of 32-byte descriptors
  - `panvk_cmd_draw`
    - `prepare_vs`
      - `prepare_vs_driver_set` allocs and inits a per-draw descriptor set
        - the set has `MAX_VS_ATTRIBS + 1 + vs->desc_info.dyn_bufs.count + vb_count`
          descriptors
        - `emit_vs_attrib` inits `MALI_ATTRIBUTE` descriptors
        - a dummy `MALI_SAMPLER` descriptor
        - `panvk_per_arch(cmd_fill_dyn_bufs)` inits `MALI_BUFFER` descriptors
          for dynamic ubos/ssbos
          - it extracts the buffer descriptors from the bound descriptor sets
            and patches in the dynamic offsets
      - `panvk_per_arch(cmd_prepare_shader_res_table)`
        - it allocs an array of `MALI_RESOURCE`s, one for each descriptor set
          (including the per-draw one)
        - the first res points to the per-draw descriptor set
        - the rest points to the user descriptor sets
      - the va of the resource array (and the array size) is written to `d0`
    - `prepare_fs`
      - `prepare_fs_driver_set` allocs and inits a per-draw descriptor set
        - the set has `1 + fs->desc_info.dyn_bufs.count` descriptors
        - a dummy `MALI_SAMPLER` descriptor
        - `panvk_per_arch(cmd_fill_dyn_bufs)` inits `MALI_BUFFER` descriptors
          for dynamic ubos/ssbos
      - `panvk_per_arch(cmd_prepare_shader_res_table)`
      - the va of the resource array (and the array size) is written to `d4`
  - `panvk_per_arch(CmdDispatchBase)`
    - `prepare_driver_set` allocs and inits a per-dispatch descriptor set
      - the first descriptor is a dummy `MALI_SAMPLER`
      - `panvk_per_arch(cmd_fill_dyn_bufs)` fills in the rest for dynamic
        ubos/ssbos
    - `panvk_per_arch(cmd_prepare_shader_res_table)`
    - the va of the resource array (and the array size) is written to `d0`
- a FAU is a Fast Access Uniform
  - each `panvk_cmd_draw` calls `prepare_push_uniforms` to prep push consts
    - `panvk_per_arch(cmd_prepare_push_uniforms)` always allocs 512 bytes
      - the first 256 bytes are for user push consts
      - the second 256 bytes are for sysvals
    - the va (and the size) is written to `d8` (for vs) and `d12` (for fs)
  - each `panvk_per_arch(CmdDispatchBase)` calls a different
    `prepare_push_uniforms` to prep push consts
    - `panvk_per_arch(cmd_prepare_push_uniforms)` always allocs 512 bytes
    - the va (and the size) is written to `d8`
  - v5 appears to use RMU, register-mapped uniform, instead
- an SPD is a `MALI_SHADER_PROGRAM`
  - `panvk_shader_upload`
    - it uploads the binary to the exec mempool
    - it allocs `MALI_SHADER_PROGRAM`
      - one for fs/cs; multiple for vs
  - each `panvk_cmd_draw` calls `prepare_vs` and `prepare_fs`
    - they write `d16`, `d18`, and `d20` to point to the spds
  - each `panvk_per_arch(CmdDispatchBase)` writes `d16` to point to the spd
- a TSD is a `MALI_LOCAL_STORAGE`
  - TLS is thread local storage
    - it is used for register spills, and its size is calculated as
      `max(shader_spill_sizes) * thread_per_core * core_count`
  - WLS is workgroup local storage?
    - it is used for glsl `shared`, cl `__local`, spirv `Workgroup` storage
  - each `panvk_cmd_draw` calls `update_tls`
    - it allocs `cmdbuf->state.tls.desc` and writes `d24` on demand
      - both `RUN_IDVS` and `RUN_FRAGMENT` use `d24` for vs and fs
    - it updates `cmdbuf->state.tls.info.tls.size`
  - each `panvk_per_arch(CmdDispatchBase)` updates TSD
    - it allocs `cmdbuf->state.tls.desc`
    - it updates `cmdbuf->state.tls.info.tls.size`
    - unlike draws, each dispatch has its own `MALI_LOCAL_STORAGE`
      - this is because each dispatch might need its own WLS
      - but it still shares the per-cmdbuf TLS storage, which is not allocated
        yet and requires additional setup
  - `panvk_per_arch(EndCommandBuffer)` calls `emit_tls`
    - it allocs the per-cmdbuf tls storage of size
      `cmdbuf->state.tls.info.tls.size`
      - this is shared by gfx and comp pipelines
    - it calls `GENX(pan_emit_tls)` to init `cmdbuf->state.tls.desc`
      - this is only used by gfx pipelines

## `RUN_IDVS`

- `MALI_CS_RUN_IDVS`
  - `flags_override` is ORed with `r56` to form the effective
    `MALI_PRIMITIVE_FLAGS`
  - `progress_increment` is always false
  - `malloc_enable` is always true
    - when idvs is used, the compiler sets `ctx->malloc_idvs` and uses a
      different instr to load varyings, which necessitates this bit
  - `draw_id_register_enable` and `draw_id`
    - when vs uses `gl_DrawID` from `VK_EXT_multi_draw`, the draw id needs to
      be preloaded for vs
    - they preload the specified value to vs r62
  - `varying_srt_select` selects `d2` instead of `d0`
  - `varying_fau_select` selects `d10` instead of `d8`
  - `varying_tsd_select` selects `d26` instead of `d24`
  - `fragment_srt_select` selects `d4` instead of `d0`
  - `fragment_tsd_select` selects `d28` instead of `d24`
  - `opcode` is `MALI_CS_OPCODE_RUN_IDVS`
- registers
  - `d0`, `d2`, `d4`, `d6`: SRT
  - `d8`, `d10`, `d12`, `d14`: FAU
  - `d16`, `d18`, `d20`, `d22`: SPD
  - `d24`, `d26`, `d28`, `d30`: TSD
  - `r32`: `Global attribute offset`, used for vertex base
    - this seems deprecated and gallium uses `r36` instead
  - `r33`: `Index count`
  - `r34`: `Instance count`
  - `r35`: `Index offset`
  - `r36`: `Vertex offset`
  - `r37`: `Instance offset`
  - `r38`: `MALI_DCD_FLAGS_2`
  - `r39`: `Index array size`
  - `d40`: `MALI_TILER_CONTEXT`
  - `d42`: `MALI_SCISSOR`, from viewport and scissor
  - `r44`: `Low depth clamp`, from viewport
  - `r45`: `High depth clamp`, from viewport
  - `d46`: `Occlusion`, va of occlusion query bo
  - `r48`: `Varying allocation` is attr count times `sizeof(vec4)`
  - `d50`: `MALI_BLEND` array
  - `d52`: `MALI_DEPTH_STENCIL`
  - `d54`: `Indices`, va of index buffer
  - `r56`: `MALI_PRIMITIVE_FLAGS`, ORed with `flags_override`
  - `r57`: `MALI_DCD_FLAGS_0`
  - `r58`: `MALI_DCD_FLAGS_1`
  - `r60`: `MALI_PRIMITIVE_SIZE`, for point size / line width
    - on v7, it is either a float (for const) or a va (for `gl_PointSize`)
    - since v9, `gl_PointSize` does not need a bo
      - it seems to use `MALI_PRIMITIVE_FLAGS` fifo
- `MALI_TILER_CONTEXT`
  - `polygon_list` is updated by the tiler (and used by `RUN_FRAGMENT`
    implicitly?)
  - `hierarchy_mask` is the allowed tile sizes
  - `sample_pattern` is sample count
  - `sample_test_disable` is always false
  - `first_provoking_vertex`
  - `fb_width`
  - `fb_height`
  - `layer_count`
  - `layer_offset`
  - `heap` is `MALI_TILER_HEAP`
  - `geometry_buffer_size` and `geometry_buffer` are the FIFO used by
    `MALI_PRIMITIVE_FLAGS`
  - `completed_top` and `completed_bottom` are updated by the tiler and used
    by `FINISH_FRAGMENT`
  - more
- `MALI_DEPTH_STENCIL`
- one or more `MALI_BLEND`

## `RUN_FRAGMENT`

- `MALI_CS_RUN_FRAGMENT`
  - `enable_tem` is always false
    - tile enable map, was used on v5 to control which tiles are preloaded?
    - since v6, frame shaders specified by DCDs (`MALI_DRAW`) are used
  - `tile_order` is always `MALI_TILE_RENDER_ORDER_Z_ORDER`
    - tile walk order, where Z is snake-like?
    - other options are row by row and col by col
  - `progress_increment` is always false
  - `opcode` is `MALI_CS_OPCODE_RUN_FRAGMENT`
- registers
  - `d40`: `MALI_FRAMEBUFFER_POINTER`
  - `d42`: `MALI_SCISSOR`, from render area
- `MALI_FRAMEBUFFER_PARAMETERS`
  - `tiler` points to per-pass `MALI_TILER_CONTEXT`
  - `frame_shader_dcds` points to `MALI_DRAW` array
  - `internal_layer_index` selects different polygon list
  - `frame_argument` is arbitrary u64 val preloaded to fs r62 and r63
  - many more
- `MALI_FRAMEBUFFER_PADDING`
- `MALI_ZS_CRC_EXTENSION` is emitted when there are zs or crc rt
  - crc rt is always disabled on panvk
    - `PAN_MESA_DEBUG=crc` enables it for transaction elimination
- one or more `MALI_RENDER_TARGET`
  - `internal_buffer_offset` is the offset of the rt in the tile memory
  - many more

## `RUN_COMPUTE`

- `MALI_CS_RUN_COMPUTE`
  - `task_increment` and `task_axis`
    - these specify how many workgroups to disatch at a time
    - e.g., if wg size is 4, wg count is 2 on the x axis, and hw has 32
      threads
      - `task_axis` can be set to `MALI_TASK_AXIS_Y`
      - `task_increment` can be set to 4
      - hw will dispatch 4 rows of wg at a time, which utilizes all 32 threads
  - `progress_increment` is always false
  - `srt_select` selects one of the four SRT regs
  - `spd_select` selects one of the four SPD regs
  - `tsd_select` selects one of the four TSD regs
  - `fau_select` selects one of the four FAU regs
  - `opcode` is `MALI_CS_OPCODE_RUN_COMPUTE`
- registers
  - `d0`, `d2`, `d4`, `d6`: SRT
  - `d8`, `d10`, `d12`, `d14`: FAU
  - `d16`, `d18`, `d20`, `d22`: SPD
  - `d24`, `d26`, `d28`, `d30`: TSD
  - `r32`: `Global attribute offset`, always 0
  - `r33`: `MALI_COMPUTE_SIZE_WORKGROUP`
  - `r34`: `Job offset X`
  - `r35`: `Job offset Y`
  - `r36`: `Job offset Z`
  - `r37`: `Job size X`
  - `r38`: `Job size Y`
  - `r39`: `Job size Z`

## Synchronization

- remember that a render pass consists of
  - `panvk_per_arch(CmdBeginRendering)`
  - `panvk_per_arch(CmdDraw*)` calls `prepare_draw`
    - `get_tiler_desc` allocs `MALI_TILER_CONTEXT` descriptors and inits
      `cmdbuf->state.gfx.render.tiler`
    - `get_fb_descs` allocs `cmdbuf->state.gfx.render.fbds`
  - `panvk_per_arch(CmdEndRendering)`
    - `flush_tiling` emits to the tiler subqueue
      - `FINISH_TILING` finishes tiling
    - `issue_fragment_jobs` emits to the fargment subqueue
      - `SYNC_WAIT64` waits on the tiler subqueue
      - `RUN_FRAGMENT` runs fragment
    - `resolve_attachments`
- remember that a vk exec dep specifies that cmds in the first sync scope
  msut happen-before cmds in the second sync scope
  - the two sync scopes are specified by `VkPipelineStageFlags2`
  - we can easily map vk graphics stages to one of the two graphics subqueues
    - `PANVK_SUBQUEUE_VERTEX_TILER`
    - `PANVK_SUBQUEUE_FRAGMENT`
  - if the two sync scopes map to the same subqueue
    - we should emit scoreboard wait to the subqueue
  - if the two sync scopes map to the different subqueues
    - we should emit scoreboard wait to the first subqueue
    - we should follow the scoreboard wait up with a seqno update
    - we should emit seqno wait to the second subqueue, to wait the first
      subqueue
- remember that a vk mem dep specifies that writes in the first access scope
  are made available, and are also made visible to reads in the second access
  scope
  - the two access scopes are specified by `VkPipelineStageFlags2` and
    `VkAccessFlags2`
  - the hw has 3 caches
    - an RW L2 shared by all gpu blocks
    - an RW L1 shared by all gpu blocks
    - an RO L1 before texture unit, rt, and zs
  - if the first access scope has any write and if the second access scope
    reads from RO L1, we should invalidate RO L1
  - if the first access scope includes cpu writes and if the second access
    scope reads from any cache, we should invalidate all caches
  - if the first access scope includes gpu writes and if the second access
    scope includes cpu reads, we should flush all caches
- `panvk_per_arch(get_cs_deps)` inits `panvk_cs_deps`
  - `needs_draw_flush` is set when there is any draw and the src stages need
    the fs results
    - this can happen when the barrier is in the middle of a render pass
    - the caller should call `panvk_per_arch(cmd_flush_draws)` to flush draws
  - `wait_sb_mask` is per-subqueue and is set when a subqueue should wait on
    prior cmds to complete
    - this is solely based on the src stages
    - the caller should call `cs_wait_slots` to emit the wait
  - `wait_subqueue_mask` is per-subqueue and is set when a subqueue should
    wait on other subqueues
    - we know which subqueues are included in the first sync scope (specified
      by the src stages)
    - if a subqueue is in the second sync scope (specified by the dst stages),
      it should wait on all other subqueues in the first sync scope
    - the caller should
      - call `cs_sync64_add` once on all subqueues in the first sync scope
      - call `cs_sync64_wait` once or more on all subqueues in the second sync
        scope
  - `cache_flush` is per-subqueue and is set when the caches should be
    flushed/invalidated
    - the caller should call `cs_flush_caches` to flush/invalidate caches
- `panvk_per_arch(get_cs_deps)` is called from
  - `panvk_per_arch(CmdPipelineBarrier2)`
  - `panvk_per_arch(CmdResetEvent2)`
  - `panvk_per_arch(CmdSetEvent2)`
  - `panvk_per_arch(CmdWaitEvents2)`

## Draw

- draw params
  - spirv `VertexIndex` is nir `vertex_id`
    - if hw only supports `vertex_id_zero_base` and `vertex_id` is lowered
      to `vertex_id_zero_base + first_vertex`
  - spirv `InstanceIndex` is nir `instance_id`
  - spirv `BaseInstance` is nir `base_instance`
  - spirv `BaseVertex` is nir `first_vertex`
    - fwiw, `gl_BaseVertex` is nir `base_vertex`, and if hw only supports
      `first_vertex`, `base_vertex` is lowered to `indexed ? first_vertex : 0`
  - spirv `DrawIndex` is nir `draw_id`
