Mesa Turnip
===========

## Device Identification

- device tree specifies `qcom,adreno-635.0`
  - msm parses `635.0` into `core`, `major`, `minor`, and `patch`
  - msm then uses the four numbers to look up the `adreno_info`
  - `MSM_PARAM_CHIP_ID` returns the 4 versions plus "speed bin" packed into
    64-bit
    - speed bin determines the max frequency
  - `MSM_PARAM_GPU_ID` returns 0
    - it is deprecated and can return numbers such as `618` on older gpus
- userspace uses chip id to look up the `fd_dev_info`
  - 635 vs 618
    - `fibers_per_sp` is `128 * 2 * 16`, not `128 * 16`
    - `reg_size_vec4` is 64, not 96
    - `instr_cache_size` is 127, not 64
    - `TPL1_DBG_ECO_CNTL` is `0x5008000`, not `0x100000`
    - these flags are set on 635
      - `supports_multiview_mask`
      - `has_z24uint_s8uint`
      - `tess_use_shared`
      - `storage_16bit`
        - `VK_KHR_16bit_storage`
      - `has_tex_filter_cubic`
        - `VK_EXT_filter_cubic`
      - `has_sample_locations`
        - `VK_EXT_sample_locations`
      - `has_lpac`
      - `has_shading_rate`
      - `has_getfiberid`
        - more `subgroupSupportedStages`
      - `has_dp2acc`
        - `integerDotProduct4x8BitPackedUnsignedAccelerated` and more
      - `has_dp4acc`
    - these flags are NOT set on 635
      - `has_cp_reg_write`
      - `has_8bpp_ubwc`
      - `ccu_cntl_gmem_unk2`
      - `indirect_draw_wfm_quirk`
      - `depth_bounds_require_depth_test_quirk`
    - `num_sp_cores` is 2, not 1
    - `num_ccu` is 2, not 1
    - `RB_UNKNOWN_8E04_blit` is both `0x00100000`
    - `PC_POWER_CNTL` is 1, not 0

## GMEM

- kernel
  - `MSM_PARAM_GMEM_BASE` is `0x100000` mostly
  - `MSM_PARAM_GMEM_SIZE` is `SZ_512K` on a618/a635
- Mesa `struct fd_dev_info`
  - `tile_align_w` and `tile_align_h`
    - alignments of the dimensions of a tile
    - 32x32 mostly
  - `gmem_align_w` and `gmem_align_h`
    - what `vkGetRenderAreaGranularity` returns
    - and what?
    - 16x4 mostly
  - `tile_max_w` and `tile_max_h`
    - max size of a tile
    - 1024x1008 mostly
  - `num_ccu` is 1 on a618 and 2 on a635
- Mesa `tu_physical_device`
  - `ccu_offset_bypass` reserves 64KB per CCU from gmem as depth cache, used
    in sysmem mode
  - `ccu_offset_gmem` reserves 16KB per CCU from gmem for msaa resolves, used
    in gmem mode
- Mesa `tu_render_pass_gmem_config`
  - `gmem_align` is 8KB mostly
  - the goal is to decide the number of gmem blocks allocated for each
    attachment
    - `ccu_offset_gmem` is 496KB mostly, thus `gmem_blocks` is 62
    - let's say att 1 has cpp 4 and att 2 has cpp 2
    - it allocates `62*4/6` to att 1 and `62*2/6` to att 2
      - that is, it allocates 41 blocks to att 1 and 21 blocks to att 2
      - ot, it allocates `41*8192/4` and `21*8192/2` pixels for att 1 and 2
      	respectively
    - `pass->gmem_pixels` is set to the max numbers of pixels that can fit
      gmem for this render pass
- Mesa `tu_tiling_config_update_tile_layout`
  - compute `fb->tile_count`, number of tiles in each direction
  - compute `fb->tile0`, dimensions of a tile
  - they must obey hw limits that are already computed and saved in render
    pass
- Mesa `tu_tiling_config_update_pipe_layout`
  - compute `fb->pipe_count`, number of pipes in each direction
  - compute `fb->pipe0`, number of tiles of a pipe covers in each direction
  - there is a total of 32 pipes; we want to use as many pipes as possible
- Mesa `tu_tiling_config_update_pipes`
  - compute pipe register values in `fb->pipe_config` and `fb->pipe_sizes`
- Mesa `tu_graphics_pipeline_create`
  - `tu_pipeline_builder_init_graphics` does a quick pass to parse info needed
    by the compiler
  - `tu_pipeline_builder_compile_shaders` compiles all shaders, including
    binning one
  - `builder->shader_iova` is initialzed with offsets of shaders in bo
  - `tu_pipeline_builder_parse_dynamic` updates `pipeline->foo_masks`, to
    indicate which states are baked and which states are dynamic
  - `tu_pipeline_builder_parse_shader_stages` emits hw cmds related to shaders
  - `tu_pipeline_builder_parse_vertex_input` emits hw cmds related to VFD
    (vertex fetch and decode)
  - `tu_pipeline_builder_parse_viewport` emits viewports and scissors, if not
    dynamic
  - and more

## Formats

- mesa formats
  - `MESA_FORMAT_<COMPONENTS>_<TYPE>`
  - `<COMPONENTS>` convention
    - packed: in LSB to MSB order
    - array: in low to high order
- pipe formats
  - `PIPE_FORMAT_<COMPONENTS>_<TYPE>`
  - `<COMPONENTS>` convention
    - packed (`is_bitmask`): in LSB to MSB order
    - array (`!is_bitmask`): in low to high order
- vk formats
  - `VK_FORMAT_<COMPONENTS>_<TYPE>`
  - `<COMPONENTS>` convention
    - packed: in MSB to LSB order
    - array: in low to high order
- msm formats
  - little-endian
  - for (MSB A R G B LSB), swap is (W X Y Z)
  - for (MSB A B G R LSB), swap is (W Z Y X)
  - in other words, swap labels each component in msb-to-lsb order
    - X means R (or D)
    - Y means G (or S)
    - Z means B
    - W means A
  - when format has 3 components, W is at the leading position
  - when format has 2 components, WZ is at the leading positions
  - when format has 1 components, WZY is at the leading positions

## turnip

- implicit fencing
  - turnip uses soft pin for all bos (i.e., their iova are known and fixed)
  - turnip keeps tracks of ALL bos allocated from a device
    - `tu_bo_init` adds the bo to `dev->bo_list`
  - all of them are marked `MSM_SUBMIT_BO_READ` and `MSM_SUBMIT_BO_WRITE`
  - when submit, all bos are added to the submit
  - in `submit_fence_sync`, the kernel driver waits for implicit fences
    - `msm_gem_sync_object` waits for existing implicit fences
  - in `submit_pin_objects`, the kernel driver pins all bos
    - `msm_gem_pin_iova` pins the pages of each bo and maps them to iommu
  - in `msm_gpu_submit`, the kernel sets up implcit fencing
    - because `MSM_SUBMIT_BO_WRITE` is always set, it calls
      `dma_resv_add_excl_fence` on all bos

## Clear & Blit

- `tu_CmdFillBuffer` uses `r2d_ops` with `r2d_clear_value`
- `tu_CmdUpdateBuffer` uploads the data to a temp buf and does a copy buffer
- `tu_CmdCopyBuffer2KHR` uses `r2d_ops` with `r2d_src_buffer`
- `tu_CmdClearColorImage` uses `r2d_ops` unless msaa
- `tu_CmdClearDepthStencilImage` is the same as above
- `tu_CmdResolveImage2KHR` uses `r2d_ops`
  - r2d does not copy msaa samples but always resolves msaa samples
- `tu_CmdCopyImage2KHR` uses `r2d_ops` unless msaa or Y8
  - not sure what's special about Y8
- `tu_CmdCopyImageToBuffer2KHR` uses `r2d_ops` unless Z24S8 or Y8
  - spec guarantees image to have 1 sample
  - not sure what's special about Z24S8
- `tu_CmdCopyBufferToImage2KHR` uses `r2d_ops` unless Z24S8 or Y8
- `tu_CmdBlitImage2KHR` uses `r2d_ops` unless...
  - msaa
  - BC1
  - cubic filter
  - z extents are different
- `tu6_clear_lrz` uses `r2d_ops` to clear lrz on `tu_CmdBeginRenderPass2`
- `tu_resolve_sysmem` uses `r2d_ops` to resolve msaa on `tu_CmdEndRenderPass2`
  and `tu_CmdNextSubpass2`
- all functions above use `cmd->cs`
- these below use `cmd->draw_cs`
  - `tu_CmdClearAttachments` can use `BLIT` event or 3D draw depending on whether
    we are in gmem or sysmem mode
  - `tu_clear_sysmem_attachment` uses `r2d_ops` to clear attachments, unless
    msaa
  - `tu_clear_gmem_attachment` uses `BLIT` event to clear GMEM on tile load
  - `tu_load_gmem_attachment` uses `BLIT` event to load GMEM on tile load
  - `tu_store_gmem_attachment` can use `BLIT` event, `r2d_ops`, or `r3d_ops`
    to blit GMEM to attachment on tile store
    - `BLIT` event is preferred but requires 16x4 alignment
    - `r2d_ops` is used unless msaa
      - `SP_PS_2D_SRC` needs iova of the source; apparently gmem is mapped and
      	has iova as well
    - `r3d_ops` appears to have bugs..

## sysmem and gmem rendering

- sysmem rendering
  - `tu6_emit_bin_size` is 0 and `BUFFERS_IN_SYSMEM`
  - `CP_SET_MARKER(RM6_BYPASS)`
  - `tu6_emit_window_scissor` is full fb
  - `tu6_emit_window_offset` is fb top-left
- gmem rendering
  - `tu6_emit_bin_size` is tile size and `BUFFERS_IN_GMEM`
  - for each tile,
    - `CP_SET_MARKER(RM6_GMEM)`
    - `tu6_emit_window_scissor` is current tile
    - `tu6_emit_window_offset` is current tile top-left
    - there is tile load and tile store

## HW binning

- `use_hw_binning` depends on the render pass
  - if the render pass exceeds hw limit, decided by `is_hw_binning_possible`,
    we have to disable hw binning
  - xfb requires hw binning if using gmem
    - sysmem always supports xfb
  - `use_hw_binning` affects `tu6_emit_tile_load`, `tu_cmd_render_tiles`, and
    `tu6_emit_tile_store`
- `tu6_emit_binning_pass` called from `tu6_tile_render_begin` starts the hw
  binning pass
  - window scissor is set to fb size
  - `CP_SET_MARKER(RM6_BINNING)` to mark hw binning mode
  - `CP_SET_VISIBILITY_OVERRIDE(1)` to ignore the vsc data
  - `CP_SET_MODE(1)` causes draw states marked `CP_SET_DRAW_STATE__0_BINNING`
    to be applied immediately?
  - `CP_WAIT_FOR_IDLE` waits for draw states to be applied?
  - `VFD_MODE_CNTL(BINNING_PASS)` to tell VFD
  - various VSC states are set
  - `PC_POWER_CNTL` is set to magic
  - `VFD_POWER_CNTL` is set to magic
  - `CP_EVENT_WRITE(UNK_2C)`
  - `RB_WINDOW_OFFSET(0, 0)`
  - `SP_TP_WINDOW_OFFSET(0, 0)`
  - finally, call `draw_cs` to start binning
  - `CP_EVENT_WRITE(UNK_2D)`
  - `CACHE_FLUSH_TS` to flush vsc data sysmem for readback by CP
  - `CP_WAIT_FOR_IDLE`
  - `CP_WAIT_FOR_ME`
  - test vsc overflow
  - `CP_SET_VISIBILITY_OVERRIDE(0)`
  - `CP_SET_MODE(0)`
- then we emit slightly different per-tile commands to use vsc data
  - `CP_SET_BIN_DATA5_*` to point to the vsc data
  - `CP_SET_VISIBILITY_OVERRIDE(0)` to use the vsc data
  - `tu6_emit_tile_load` has slightly different commands, mainly to skip
    loading when the tile is not covered
  - `CP_SET_MARKER(RM6_ENDVIS)` after `draw_cs`
  - `tu6_emit_tile_store` has slightly different commands, mainly to skip
    storing when the tile is not covered

## `VkAccessFlagBits`

- these are from CP and are uncached
  - `VK_ACCESS_INDEX_READ_BIT`
  - `VK_ACCESS_INDIRECT_COMMAND_READ_BIT`
  - `VK_ACCESS_TRANSFORM_FEEDBACK_COUNTER_READ_BIT_EXT`
  - `VK_ACCESS_TRANSFORM_FEEDBACK_COUNTER_WRITE_BIT_EXT`
  - `VK_ACCESS_CONDITIONAL_RENDERING_READ_BIT_EXT`
- these are from VFD and are UCHE cached
  - `VK_ACCESS_VERTEX_ATTRIBUTE_READ_BIT`
- these are from VPC and are UCHE cached
  - `VK_ACCESS_TRANSFORM_FEEDBACK_WRITE_BIT_EXT`
- these are from SP and are UCHE (or L1) cached
  - `VK_ACCESS_UNIFORM_READ_BIT`
  - `VK_ACCESS_INPUT_ATTACHMENT_READ_BIT`
  - `VK_ACCESS_SHADER_READ_BIT`
  - `VK_ACCESS_SHADER_WRITE_BIT`
  - `VK_ACCESS_TRANSFER_READ_BIT`
    - `CP_BLIT` is how we transfer in most cases, and is equivalent to sysmem
      rendering
    - `CP_DRAW` is a fallback, and is also equivalent to sysmem rendering
    - it is thus L1 cached for `TRANSFER_READ`
- these are from RB and can be CCU cached or uncached
  - `VK_ACCESS_COLOR_ATTACHMENT_READ_BIT`
  - `VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT`
  - `VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT`
  - `VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT`
  - `VK_ACCESS_TRANSFER_WRITE_BIT`
    - this is equivalent to sysmem rendering and is CCU cached
    - see `VK_ACCESS_TRANSFER_READ_BIT` above for details
  - in sysmem rendering, they are CCU cached
  - in gmem rendering, there are tile load (READ) and tile store (WRITE)
    - tile load uses event `BLIT` and is uncached
      - not UCHE or L1 cached?
    - tile store can use
      - event `BLIT` and is CCU cached
        - we always `PC_CCU_RESOLVE_TS` implicitly so it can be considered
          uncached
      - `CP_BLIT` or `CP_DRAW`, and is CCU cached
        - we always `PC_CCU_FLUSH_COLOR_TS` implicitly so it can be considered
          uncached
        - note in these cases, we texture from gmem.  We always
          `CACHE_INVALIDATE` implicitly first.
  - when we are inside of a render pass, we know whehter it is gmem or sysmem
    rendering
  - when we are outside of a render pass, we assume sysmem rendering
    - we might unnecessarily flush/invalidate CCU, but the result will be
      correct
    - is that what `pending_flush_bits` is for, to avoid the unnecessary
      CCU flush/invalidate?
- these are from host and may or may not be coherent
  - `VK_ACCESS_HOST_READ_BIT`
  - `VK_ACCESS_HOST_WRITE_BIT`
  - it can be uncached and thus coherent
  - it can also be CPU cached and not coherent
  - or it can be CPU cached but coherent
    - when GPU snoops or the cache is an LLC
- these map to other access flags
  - `VK_ACCESS_MEMORY_READ_BIT`
  - `VK_ACCESS_MEMORY_WRITE_BIT`
- `vk2tu_access`
  - `TU_ACCESS_UCHE_READ` if UCHE (or L1) cached reads
  - `TU_ACCESS_UCHE_WRITE` if UCHE cached writes
  - `TU_ACCESS_CCU_COLOR_INCOHERENT_READ` if CCU color cached reads
    - `TU_ACCESS_CCU_COLOR_READ` is unused
  - `TU_ACCESS_CCU_COLOR_INCOHERENT_WRITE` if CCU color cached writes
    - `TU_ACCESS_CCU_COLOR_WRITE` if transfer writes
  - `TU_ACCESS_CCU_DEPTH_INCOHERENT_READ` if CCU depth cached reads
  - `TU_ACCESS_CCU_DEPTH_INCOHERENT_WRITE` if CCU depth cached writes
  - `TU_ACCESS_CCU_DEPTH_READ`
  - `TU_ACCESS_CCU_DEPTH_WRITE`
  - `TU_ACCESS_SYSMEM_READ`
  - `TU_ACCESS_SYSMEM_WRITE`
    - except for CP writes, where we use `TU_ACCESS_CP_WRITE` instead

## Cache Management

- `tu_emit_cache_flush_*` emits cache flush/invalidate commands
  - `tu_emit_cache_flush_ccu` is called with `cmd->cs` in
    - `tu6_sysmem_render_begin`
    - `tu6_tile_render_begin`
    - various clear/blit vk commands that are outside of render pass
  - `tu_emit_cache_flush` is called with `cmd->cs`, or
    `tu_emit_cache_flush_renderpass` is called with `cmd->draw_cs`,
    depending on whether outside or inside of render pass, in
    - `tu_EndCommandBuffer`
    - `tu_CmdExecuteCommands`
    - `tu_CmdBeginConditionalRenderingEXT`
    - `tu_CmdWriteBufferMarkerAMD`
  - `tu_emit_cache_flush` is called with `cmd->cs` in
    - `tu_dispatch`
    - `write_event`
  - `tu_emit_cache_flush_renderpass` is called with `cmd->draw_cs` in
    - `tu6_draw_common`
    - `tu_CmdClearAttachments`
- in other words, in a simple cmdbuf with a simple render pass and a simple
  draw, we have these
  - `tu_cache_init` sets `flush_bits` to 0 and `pending_flush_bits` to all
    invalidates
  - `tu6_init_hw` removes `TU_CMD_FLAG_CACHE_INVALIDATE` from
    `pending_flush_bit`
  - `tu_CmdBeginRenderPass2`
    - calls `tu_subpass_barrier` with the default dep defined by the spec
      - `src_flags` is 0
      - `dst_flags` is `TU_ACCESS_UCHE_READ` and
        `TU_ACCESS_CCU_{COLOR,DEPTH}_INCOHERENT_{READ,WRITE}`
      - `tu_flush_for_access`
	- moves the two invalidate bits from `pending_flush_bits` to
	  `flush_bits`
      - `src_stage` is `TU_STAGE_CP`
      - `dst_stage` is `TU_STAGE_CP`
      - `tu_flush_for_stage`
	- adds `TU_CMD_FLAG_WAIT_FOR_IDLE` to `flush_bits`
	- adds `TU_CMD_FLAG_WAIT_FOR_ME` to `pending_flush_bits`
    - propagates down `pending_flush_bit` to render pass
  - `tu_Draw` calls `tu_emit_cache_flush_renderpass`
    - no-op because render pass's `flush_bits` is 0
  - `tu_CmdEndRenderPass2`
    - `tu_emit_cache_flush_ccu` called by render begin
      - because switching from sysmem to gmem, CCU is entire
      	flushed/invalidated 
      - `flush_bits` is 0
      - `pending_flush_bits` still has `TU_CMD_FLAG_WAIT_FOR_ME`
    - propogates render pass `pending_flush_bit` back
      - no change
    - `tu_subpass_barrier`
      - `src_flags` is `TU_ACCESS_UCHE_READ` and
        `TU_ACCESS_CCU_{COLOR,DEPTH}_INCOHERENT_{READ,WRITE}`
      - `dst_flags` is 0
      - `tu_flush_for_access`
	- sets both CCU flushes in `flush_bits`
      - `src_stage` is `TU_STAGE_PS`
      - `dst_stage` is `TU_STAGE_PS`
      - `tu_flush_for_stage` is no-op
  - `tu_EndCommandBuffer`
    - `tu_flush_all_pending` is no-op
    - `tu_emit_cache_flush` flushes CCU entirely
- walking through the example above was so boring
- I guess
  - `pending_flush_bits` are what need to happen
  - `flush_bits` defers command emissions
  - when we see a barrier, explicit or implicit, we calculate the flush bits
    from the barrier first.  We then AND them with `pending_flush_bits` and
    move the resulting bits to `flush_bits`
  - only when `tu_emit_cache_flush_*`, we emits the commands and clears
    `flush_bits`
