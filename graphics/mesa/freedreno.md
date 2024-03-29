Mesa Freedreno
==============

## `fd_device` and `fd_pipe`

- `fd_device_new` creates an `fd_device` from a drm fd
  - depending on the drm fd, it calls `msm_device_new` or `virtio_device_new`
- `fd_pipe_new2` creates an `fd_pipe` for each `pipe_context`
  - depending on whether softpin is supported, it uses `sp_funcs` or
    `legacy_funcs`
  - internally it creates a msm submitqueue

## `fd_ringbuffer`

- `fd_submit_new` and `fd_submit_del` are called for each batch buffer
- `fd_submit_new_ringbuffer` creates a ringbuffer for short-lived cmdstream
  - it calls `fd_submit_sp_new_ringbuffer` on newer kernel
  - it can allocates a new bo (`fd_bo_new_ring`) or it can suballocate from
    `submit->suballoc_ring` (`fd_submit_suballoc_ring_bo`)
- `fd_ringbuffer_new_object` creates a ringbuffer for long-lived stateobj
  - it calls `fd_ringbuffer_sp_new_object` on newer kernel
  - internally, it suballocates from a 32K (`SUBALLOC_SIZE`)
    `dev->suballoc_bo`
  - `fd_ringbuffer_sp` has a dynamic array of `fd_cmd_sp` and can suballocate
    more when it runs out of space
- `OUT_RELOC` adds an `fd_reloc` to an `fd_ringbuffer`

## `fd_batch`

- a `fd_batch` has several `fd_ringbuffer`s
  - `gmem`: IB1
  - `prologue`: IB2
  - `epilogue`: IB2
  - `draw`: IB2
- most commands are recorded into `draw`
- when the time comes to flush, `fd_gmem_render_tiles` emits to `gmem` and
  submits

## Batch Reordering

- app draws to a texture and draws the texture to rt
  - for an imr, it can draw to the texture and then draw to the rt
  - as a tiler, we would like to consider them as a renderpass to draw to gmem
    and to sample from gmem.  The renderpass is not ended until flush.
- `ctx->batch`
  - upon fb change, `fd_set_framebuffer_state` resets `ctx->batch`
  - `fd_context_batch` returns `ctx->batch`
    - if it is null, `fd_batch_from_fb` looks up the batch for the fb
  - iow, `ctx->batch` is the current batch for the current fb, and there can
    be multiple batches before flushing
- `fd_context_flush`
  - `fd_bc_flush` calls `fd_batch_flush` on all batches
    - this is like ending a renderpass
    - `fd_gmem_render_tiles` and `fd_submit_flush` are called for each batch
  - `fd_submit_flush` calls `fd_submit_sp_flush` on modern kernels
    - it defers submits such that it can merge them
    - `flush_deferred_submits` submits all deferred submits at once to the `sq`
      thread pool
- `fd_pipe_fence_finish`
  - it calls `fence_flush` to flush deferred submits
    - `fd_fence_flush` calls `fd_pipe_sp_flush` on modern kernels
- when combined with tc (threaded context), things can be a bit annoying
  - when the main thread `glFlush`, mesa often calls `tc_flush` with
    `PIPE_FLUSH_ASYNC`.  Prior gallium commands and the pipe flush is queued
    to tc's `gdrv` thread
  - if the main thread renders the next frame and does another `glFlush`, it
    can still be queued up for `gdrv`
  - when `gdrv` runs, it calls `fd_submit_sp_flush` twice and could defer both
    submits
  - the two submits will be submitted to the kernel together and will be
    signaled together

## GMEM Layout

- `calc_nbins`
  - tile max: 1024x1008
  - tile align: 16x4
  - the goal is to minimize bin count
  - algorithm
    - calculate minimal bin count horizontally
    - calculate minimal bin count verticall
    - while bin size cannot fit in gmem
      - increment vertical or horizontal bin count
    - see if we can make a small adjustment to do better

## Freedreno

- `fd_draw_vbo` emits into `batch->draw` using `fd6_draw_vbo`
  - `fd6_emit_state`
    - check dirty states and create (or reference pre-created) state objects (small IBs)
      - `FD6_GROUP_VBO`
      - `FD6_GROUP_VBO_BINNING`
      - `FD6_GROUP_ZSA`
      - `FD6_GROUP_LRZ`
      - `FD6_GROUP_LRZ_BINNING`
      - `FD6_GROUP_PROG`
      - `FD6_GROUP_PROG_BINNING`
      - `FD6_GROUP_RASTERIZER`
      - `FD6_GROUP_VS_CONST`
      - `FD6_GROUP_VS_TEX`
      - `FD6_GROUP_FS_TEX`
    - emit many other non-stateobj states
    - `SET_DRAW_STATE()` to reference the stateobjs
      - the opcode specifies a set of IBs and a set of masks saying when are the IBs applied (binning or rendering)
        - the IBs are probably cached in an internal memory
        - so that HW can execute different IBs in different states (binding or rendering)
      - each IB has a key to allow partial updates
        - update this IB in the internal cache but not others
  - `REG_WRITE(VFD_INDEX_OFFSET)`: first index
  - `REG_WRITE(PC_RESTART_INDEX)`: primitive restart
  - `emit_marker6()`: for debugging
  - `DRAW_INDEX_OFFSET(draw_patches, instance_count, count)`
  - `emit_marker6()`: for debugging
- `fd_batch_flush` queues jobs into `flush_queue` thread
  - `fd_gmem_render_tiles` does the jobs
    - sysmem mode: bypassing GMEM and is used for very simple operations (blits)
      - `emit_sysmem_prep`
      - `emit_ib`
      - `emit_sysmem_fini`
    - gmem mode:
      - `emit_tile_init`
      - for each tile
        - `emit_tile_prep`
        - `emit_tile_mem2gmem`
        - `emit_tile_renderprep`
        - `emit_ib`
        - `emit_tile_gmem2mem`
      - `emit_tile_fini`
- pipeline
  - VFD: vertex fetch and decode
  - SP: stream processor
  - VPC: to store vs outout
  - GRAS: rasterizer
  - RB: render buffer?
  - RBBM
  - CP
  - PC
  - HLSQ: shader resoureces?
  - VBIF
  - VSC
- VFD (vertex fetch and decode)
  - `set_vertex_buffer`
    - `REG_WRITE(VFD_FETCH)` and friends in `FD6_GROUP_VBO`
  - `bind_vertex_element_states`
    - `REG_WRITE(VFD_DECODE)` and friends in `FD6_GROUP_VBO`
- VS
  - `set_clip_state`
    - baked into VS
  - `set_constant_buffer`
    - `CP_LOAD_STATE6_GEOM(offset=slot, SB6_VS_SHADER, ST6_CONSTANTS)` in `FD6_GROUP_VS_CONST`
  - `set_shader_buffers`
    - `CP_LOAD_STATE6_GEOM(SB6_SSBO)` in draw IB
    - `CP_LOAD_STATE6_GEOM(SB6_VS_SHADER)` in draw IB, to describe buffer sizes
  - `bind_sampler_states` and `set_sampler_views`
    - save to `ctx->tex[shaderStage]` first and process later in `fd6_emit_textures`
    - convert sampler states to an array of (4-dword `SAMPLER_STATE`) in BO
    - convert sampler views to an array of (16-dword `SAMPLER_VIEW`) in BO
    - in `FD6_GROUP_VS_TEX` IB,
      - `CP_LOAD_STATE6_GEOM(SB6_VS_TEX)` to load both arrays
      - `REG_WRITE(P_VS_TEX_SAMP_LO)` and friends
  - `bind_vs_state`
    - in creation, NIR IR is stored
      - HW instr is not generated until the "variant" is known
    - `CP_LOAD_STATE6_GEMO(offset=0, SB6_VS_SHADER, ST6_SHADER, indirect)` to load the consts
    - `fd6_program_create` looks at all bs (binning shader), vs, and fs to create a stateobj
      - `REG_WRITE(SP_VS_CONFIG)`
      - `REG_WRITE(HLSQ_VS_CNTL)`
      - `REG_WRITE(SP_VS_CTRL_REG0)`
      - `REG_WRITE(VPC_VAR_DISABLE)`
      - `REG_WRITE(SP_VS_OUT_REG)`
      - `REG_WRITE(SP_VS_VPC_DST_REG)`
      - `REG_WRITE(SP_VS_OBJ_START_LO)`
      - `CP_LOAD_STATE6_GEOM()` to load VS
      - `REG_WRITE(SP_PRIMITIVE_CNTL)`
- GRAS (gemoetry rasterizer?)
  - `set_viewport_states`
    - `REG_WRITE(GRAS_CL_VPORT_XOFFSET_0)` and friends in draw IB
  - `bind_rasterizer_state`
    - in state creation, an IB is created to `REG_WRITE(GRAS_SU_CNTL)` and friends
    - in bind, the IB is used as `FD6_GROUP_RASTERIZER`
    - the state also interacts with other states
  - `set_scissor_states`
    - `REG_WRITE(GRAS_SC_WINDOW_SCISSOR_TL)` and friends in gmem IB
    - affects binning
- VSC (binning / visibility stream c?)
- FS
  - see VS
  - `set_shader_images`
    - `CP_LOAD_STATE6_FRAG(offset=slot, SB6_FS_TEX, ST6_CONSTANTS, direct)` for use by imageLoad
    - `CP_LOAD_STATE6_FRAG(offset=slot, SB6_SSBO, ST6_2, direct) `for use by imageStore
    - `CP_LOAD_STATE6_FRAG(SB6_FS_SHADER) `to describe image dimensions
- RB (render buffer?)
  - `set_blend_color`
    - `REG_WRITE(RB_BLEND_RED_F32)` and friends in draw IB
  - `set_stencil_ref`
    - `REG_WRITE(RB_STENCILREF)` in draw IB
  - `bind_blend_states`
    - `REG_WRITE(RB_MRT_CONTROL)` and friends in draw IB
  - `bind_depth_stencil_alpha_state`
    - in state creation, an IB is created to `REG_WRITE(RB_ALPHA_CONTROL)` and friends
    - in bind, the IB is used as `FD6_GROUP_ZSA`
  - `set_framebuffer_state`

## Freedreno `render_tiles`

- `fd6_emit_tile_init`
  - `fd6_emit_restore`
    - `fd6_cache_flush`
      - `EVENT_WRITE(0x31)` to trigger cache flush
    - `REG_WRITE(HLSQ_UPDATE_CNTL, 0xfffff)`
    - `REG_WRITE(a bunch of registers)`
    - `emit_marker6`
      - `REG_WRITE(SCRATCH_REG7, ++marker_cnt)`
    - `REG_WRITE(some more registers)`
    - `SET_DRAW_STATE()` ?
    - `REG_WRITE(a bunch more regs)`
  - `fd6_emit_lrz_flush`
    - `EVENT_WRITE(LRZ_FLUSH)`
  - `fd6_cache_flush`
  - `SKIP_IB2_ENABLE_GLOBAL(0)`
  - `fd_wfi`
    - `WAIT_FOR_IDLE()`
  - `REG_WRITE(RB_CCU_CNTL, 0x7c40004)`
  - `emit_zs`
    - `REG_WRITE(RB_DEPTH_BUFFER_INFO, ...)`
    - `REG_WRITE(GRAS_SU_DEPTH_BUFFER_INF, ...)`
    - `REG_WRITE(GRAS_LRZ_BUFFER_BASE_LO, ...)`
    - `REG_WRITE(RB_STENCIL_INFO, ...)`
  - `emit_mrt`
    - for each render target
      - `REG_WRITE(RB_MRT_BUF_INFO, ...)`: addr, format, tiling, stride
      - `REG_WRITE(SP_FS_MRT_REG)`: format
    - `REG_WRITE(RB_SRGB_CNTL)`: sRGB rendering
    - `REG_WRITE(SP_SRGB_CNTL)`: sRGB rendering
    - `REG_WRITE(SRB_RENDER_COMPONENTS)`: which MRTs are on
    - `REG_WRITE(SP_FS_RENDER_COMPONENTS)`: which MRTs are on
  - `disable_msaa`
    - `REG_WRITE(SP_TP_RAS_MSAA_CNTL, ...)`;
    - `REG_WRITE(GRAS_RAS_MSAA_CNTL, ...)`;
    - `REG_WRITE(RB_RAS_MSAA_CNTL, ...)`;
  - `set_bin_size(BINNING_PASS)`
    - `REG_WRITE(GRAS_BIN_CONTROL)`: bin width/height
    - `REG_WRITE(RB_BIN_CONTROL)`: bin width/height
    - `REG_WRITE(RB_BIN_CONTROL2)`: bin width/height
  - `update_render_cntl(true)`
    - `CP_ONE_REG_WRITE(RB_RENDER_CNTL, ...)`
  - `emit_binning_pass`
    - `set_scissor`
      - `REG_WRITE(GRAS_SC_WINDOW_SCISSOR_TL)`
      - `REG_WRITE(GGRAS_RESOLVE_CNTL_1)`
    - `emit_marker6`
    - `SET_MARKER(RM6_BINNING)`: switch to binning mode
    - `emit_marker6`
    - `SET_VISIBILITY_OVERRIDE(1)`
    - `SET_MODE(1)`
    - `WAIT_FOR_IDLE()`
    - `REG_WRITE(VFD_MODE_CNTL, BINNING_PASS)`
    - `update_vsc_pipe`
      - `REG_WRITE(VSC_BIN_SIZE, ...)`
      - `REG_WRITE(VSC_BIN_COUNT)`
      - `REG_WRITE(VSC_PIPE_CONFIG_REG, ...)`
      - `REG_WRITE(VSC_PIPE_DATA2_ADDRESS_LO, ...)`
      - `REG_WRITE(VSC_PIPE_DATA_ADDRESS_LO, ...)`
    - `REG_WRITE(unknown regs)`
    - `EVENT_WRITE(0x2c)`
    - `REG_WRITE(RB_WINDOW_OFFSET)`
    - `REG_WRITE(SP_TP_WINDOW_OFFSET)`
    - `fd6_emit_ib(batch->draw)`
      - `emit_marker6`
      - `INDIRECT_BUFFER(address, size)`
      - `emit_marker6`
    - `SET_DRAW_STATE()`
    - `EVENT_WRITE(0x2d)`
    - `EVENT_WRITE(CACHE_FLUSH_TS, blit_mem, 0x0)`
    - `fd_wfi`
  - `set_bin_size(USE_VIZ)`
  - `REG_WRITE(VFD_MODE_CNTL, 0)`
  - `update_render_cntl(false)`
- for each tile
  - `fd6_emit_tile_prep`
    - `SET_MARKER(MODE(0x7))`
    - `emit_marker6`
    - `SET_MARKER(RM6_GMEM)`
    - `emit_marker6`
    - `set_scissor`
    - `set_window_offset`
      - `REG_WRITE(RB_WINDOW_OFFSET)`
      - `REG_WRITE(RB_WINDOW_OFFSET2)`
      - `REG_WRITE(SP_WINDOW_OFFSET)`
      - `REG_WRITE(SP_TP_WINDOW_OFFSET)`
    - `REG_WRITE(VPC_SO_OVERRIDE, DISABLE)`
    - `WAIT_FOR_ME()`
    - `SET_VISIBILITY_OVERRIDE(0)`
    - `SET_MODE(0)`
    - `SET_BIN_DATA5()`
  - `fd6_emit_tile_mem2gmem`
    - no-op
  - `fd6_emit_tile_renderprep`
    - `fd6_emit_ib(batch->tile_setup)`
  - `fd6_emit_ib(batch->draw)`
  - `fd6_emit_tile_gmem2mem`
    - `fd6_emit_ib(batch->tile_fini)`
- `fd6_emit_tile_fini`
  - `REG_WRITE(GRAS_LRZ_CNTL, ENABLE | UNK3)`
  - `EVENT_WRITE(LRZ_FLUSH)`
  - `fdm_event_write(CACHE_FLUSH_TS)`
    - `EVENT_WRITE(CACHE_FLUSH_TS, blit_mem, seqno)`
- This is it.  Let's visit IBs
- `fd6_clear_lrz` creates `batch->lrz_clear`
  - `emit_marker6()`
  - `SET_MARKER(RM6_BYPASS)`
  - `emit_marker6()`
  - `REG_WRITE(RB_CCU_CNTL, 0x10000000)`
  - `emit_marker6()`
  - `SET_MARKER(0xc)`
  - `emit_marker6()`
  - `REG_WRITE(unknown)`
  - `REG_WRITE(SP_PS_2D_SRC_INFO)`
  - `REG_WRITE(a bunch more)`
  - `REG_WRITE(RB_2D_DST_INFO)`
  - `REG_WRITE(more)`
  - `EVENT_WRITE(0x3f)`
  - `WAIT_FOR_IDLE()`
  - `REG_WRITE(unknown)`
  - `BLIT(SCALE)`
  - `REG_WRITE(unknown)`
  - `EVENT_WRITE(0x1d)`
  - `EVENT_WRITE(FACENESS_FLUSH)`
  - `EVENT_WRITE(CACHE_FLUSH_TS)`
  - `fd6_cache_flush`
- `CP_BLIT`: sys mem to sys mem 2D blit
- `EVENT_WRITE(BLIT)`: sys mem <-> gmem blit
  - GMEM mode (GMEM is dest): sys -> gmem
  - RESOLVE mode (GMEM is src): gmem -> sys
  - `CP_SET_MARKER` sets the mode
  - `RB_BLIT_xxx` to set up sys mem address
