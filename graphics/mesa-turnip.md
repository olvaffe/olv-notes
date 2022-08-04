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
  - `FMT6_<COMPONENTS>_<TYPE>`
  - `<COMPONENTS>` convention
    - although not explicitly mentioned, they are usually in RGBA order
      - `FMT6_5_5_5_1_UNORM` means R5, G5, B5, and A1
      - there are exceptions
        - `FMT6_1_5_5_5_UNORM` also means R5, G5, B5, A1
          - it should probably be named `FMT6_5_5_5_1_UNORM_REV`
    - component order in memory is decided by swap
  - swap
    - one of `WZYX`, `WXYZ`, `ZYXW`, or `XYZW`
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
  - examples
    - `FMT6_5_5_5_1_UNORM` means R5, G5, B5, and A1
    - with WXYZ, we have (MSB A1 R5 G5 B5 LSB), which is
      `PIPE_FORMAT_B5G5R5A1_UNORM`
    - with XYZW, we have (MSB R5 G5 B5 A1 LSB), which is
      `PIPE_FORMAT_A1B5G5R5_UNORM`
    - `FMT6_1_5_5_5_UNORM` and its swaps make no sense
      - I guess it expands (MSB R5 G5 B5 A1 LSB) into (MSB A8 R8 G8 B8 LSB)
      	first
      - with that, WXYZ returns the correct color
      - WZYX swaps R and B
- some pipe formats are used only internally by certain drivers
  - for example, for all but turnip, the Y plane of NV12 is `PIPE_FORMAT_R8_UNORM`
  - for turnip, it is `PIPE_FORMAT_Y8_UNORM`
  - this allows turnip to map R8 to `FMT6_8_UNORM` and Y8 to `FMT6_NV12_Y`
  - the reason is that, when compressed (or tiled?), the two formats are
    different
- interesting facts about the format table
  - `SSCALED` and `USCALED` are only for vertices, and
    - both `PIPE_FORMAT_*_UINT` and `PIPE_FORMAT_*_USCALED` are mapped to
      `FMT6_*_UINT`
    - boith `PIPE_FORMAT_*_SINT` and `PIPE_FORMAT_*_SSCALED` are mapped to
      `FMT6_*_SINT`
    - there is a bit in `A6XX_VFD_DECODE_INSTR` that is based on
      `vk_format_is_int` 
  - both `PIPE_FORMAT_*_UNORM` and `PIPE_FORMAT_*_SRGB` are mapped to `FMT6_*_UNORM`
  - `PIPE_FORMAT_A8_UNORM` is mapped to `FMT6_8_UNORM` for texturing and
    `FMT6_A8_UNORM` for rendering
  - `PIPE_FORMAT_R10G10B10A2_UNORM` is mapped to `FMT6_10_10_10_2_UNORM` for
    texturing and `FMT6_10_10_10_2_UNORM_DEST` for rendering
  - are the mappings for 5551/1555 correct?
    - yes, `FMT6_5_5_5_1_UNORM` and swap are correct
    - `FMT6_1_5_5_5_UNORM` and its swaps make no sense. Accept it.
  - depth is complicated
    - depth buffers use `DEPTH6_*` instead of `FMT6_*`
    - depth formats are still mapped to `FMT6_*` in the format table because
      of
      - texturing
      - clear/copy/blit dst
    - `PIPE_FORMAT_S8_UINT` is mapped to `FMT6_8_UINT`
    - `PIPE_FORMAT_Z16_UNORM` is mapped to `FMT6_16_UNORM`
    - `PIPE_FORMAT_Z24X8_UNORM` is mapped to `FMT6_Z24_UNORM_S8_UINT`
      - this is the mapping in the format table, and is only used for
      	texturing
      - for blitting, `tu6_base_format` forces `FMT6_8_8_8_8_UNORM` and
      	adjusts the clear value; if ubwc, forces
      	`FMT6_Z24_UNORM_S8_UINT_AS_R8G8B8A8`
      - this should be because the 2d engine cannot do 24-bit but uses
      	`R2D_UNORM8`
    - `PIPE_FORMAT_X24S8_UINT` is mapped to `FMT6_8_8_8_8_UINT`
      - this can be returned by `tu_format_for_aspect`
    - `PIPE_FORMAT_Z24_UNORM_S8_UINT` is mapped to `FMT6_Z24_UNORM_S8_UINT`
    - `PIPE_FORMAT_Z32_FLOAT` is mapped to `FMT6_32_FLOAT`
    - `PIPE_FORMAT_Z32_FLOAT_S8X24_UINT` is mapped to `FMT6_32_FLOAT`
    - `PIPE_FORMAT_X32_S8X24_UINT` is mapped to `FMT6_8_UINT`
    - `PIPE_FORMAT_Z24_UNORM_S8_UINT_AS_R8G8B8A8` is mapped to
      `FMT6_Z24_UNORM_S8_UINT_AS_R8G8B8A8`
- vertex buffers
  - `tu6_emit_vertex_input` emits the vertex inputs
  - it calls `fd6_vertex_format`
    - `fd6_vertex_format` for format
    - `fd6_vertex_swap` for swap
- color buffers
  - `tu6_emit_mrt` emits the color buffers
  - it uses `fdl6_view::RB_MRT_BUF_INFO` initialized by `fdl6_view_init`
    - `fd6_color_format` for format
    - `fd6_color_swap` for swap
- depth buffers
  - `tu6_emit_zs` emits the depth buffer
  - it calls `tu6_pipe2depth` to get the format
    - one of `DEPTH6_16`, `DEPTH6_24_8`, or `DEPTH6_32`
    - no swap
    - `fd6_pipe2depth` is unused
  - there is also hw support for separate stencil
- textures
  - `write_image_descriptor` writes the descriptor
  - it uses `fdl6_view::descriptor` initialized by `fdl6_view_init`
    - `fd6_texture_format` for format
    - `fd6_texture_swap` for swap

## Image Layout

- msaa
  - samples of a pixel are packed together in memory
- 1D and 2D images
  - a non-array image has its miplevels arranged one after another in memory
  - an array image is an array of non-array images in memory
  - this is called "layer first"
- 3D images
  - all slices of a miplevel are arranged one after another in memory
  - then miplevels are arranged one after another in memory
- UBWC
  - block size
    - 16x8 for R8G8
    - 32x8 for Y8
    - 16x4 for cpp 1, 2, and 4
    - 8x4 for cpp 8
    - 4x4 for cpp 16
    - 4x2 for cpp 32
  - one byte per block
  - "layer first"
  - we also put UBWC data before image data
- `fdl_pitch` returns the row pitch for a miplevel
- `fdl_layer_stride` returns the layer pitch for an 1D/2D array or a 3D image
- `fdl_tile_mode` returns whether a miplevel is tiled
  - if ubwc or depth/stencil, all miplevels are tiled
  - otherwise, miplevels become linear when their widths are less than 16,
    `FDL_MIN_UBWC_WIDTH`

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

## Queues

- `vkQueueSubmit`
  - msm supports binary syncobjs but not timeline syncobjs
  - the runtime picks `VK_DEVICE_TIMELINE_MODE_EMULATED` and emulates
    timeline syncobjs using binary syncobjs
  - to handle wait-before-signal, the runtime might reorder the submits
- `tu_queue_submit`
  - each `tu_cs` has a list of `tu_cs_entry`, which are regions of `tu_bo`s
  - all `tu_cs_entry` from all `tu_cmd_buffer::cs`s are collected into `struct
    tu_queue_submit`
  - `DRM_MSM_GEM_SUBMIT` submits them to the kernel

## Command Buffers

- a `tu_cmd_buffer` has several `tu_cs`s
  - `cs` is what gets submitted to the kernel directly
  - `draw_cs` is for commands inside a render pass
    - in sysmem, it is called once
    - in gmem, it is called for binning and for each tile
  - `tile_store_cs` is for tile resolve and is called for each tile
  - `draw_epilogue_cs` is for queries in a render pass and is called at the
    end of the render pass
  - `sub_cs` is for suballocating stateobjs and inline data
- `sub_cs`
  - the inline data for `vkCmdUpdateBuffer`
  - temp image desc for 3D blit src
  - temp image descs for input attachments
  - stateobj for vertex buffers
    - `TU_CMD_DIRTY_VERTEX_BUFFERS` and `TU_DRAW_STATE_VB`
    - `TU_CMD_DIRTY_VB_STRIDE` and `TU_DYNAMIC_STATE_VB_STRIDE`
  - dynamic descs for a descriptor set
  - stateobj for descriptor sets
  - stateobjs for dynamic states such as
    - scissors and viewports
    - line width, cull mode
    - depth bias and bounds
    - stencil masks and refs
    - blend consts, logic op, etc.
  - inline data for push consts
  - stateobj for lrz
  - stateobj for draw info
- action commands outside of a render pass go to `cs`
  - `vkCmdCopy*`, `vkCmdClear*`, and friends (except `vkCmdClearAttachments`)
  - `vkCmdSetEvent` and friends
  - `vkCmd*Query*` when outside of a render pass
  - `vkCmdDispatch`
- action commands inside of a render pass go to `draw_cs`
  - `vkCmdClearAttachments`
- state commands outside or inside of a render pass go to `draw_cs`
  - because we need to reset the states for each tile
- `draw_epilogue_cs`
  - for queries
- barriers 
  - barriers are combined and deferred
  - they go to `draw_cs` in `tu_emit_cache_flush_renderpass` normally
  - in some cases, they go to `cs` in `tu_emit_cache_flush`
- execute secondary command buffers
- render pass begin/end

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

## `VkPipelineStageFlagBits2`

- these are CP
  - `VK_PIPELINE_STAGE_2_CONDITIONAL_RENDERING_BIT_EXT`
  - `VK_PIPELINE_STAGE_2_DRAW_INDIRECT_BIT`
  - `VK_PIPELINE_STAGE_2_INDEX_INPUT_BIT`
- these are VFD
  - `VK_PIPELINE_STAGE_2_VERTEX_ATTRIBUTE_INPUT_BIT`
- these are `SP_{VS,HD,DS,GS}` respectively
  - `VK_PIPELINE_STAGE_2_VERTEX_SHADER_BIT`
  - `VK_PIPELINE_STAGE_2_TESSELLATION_CONTROL_SHADER_BIT`
  - `VK_PIPELINE_STAGE_2_TESSELLATION_EVALUATION_SHADER_BIT`
  - `VK_PIPELINE_STAGE_2_GEOMETRY_SHADER_BIT`
- these are PC
  - NA
- these are VPC
  - `VK_PIPELINE_STAGE_2_TRANSFORM_FEEDBACK_BIT_EXT`
- these are GRAS
  - `VK_PIPELINE_STAGE_2_EARLY_FRAGMENT_TESTS_BIT`
- these are `SP_FS`
  - `VK_PIPELINE_STAGE_2_FRAGMENT_SHADER_BIT`
- these are RB
  - `VK_PIPELINE_STAGE_2_LATE_FRAGMENT_TESTS_BIT`
  - `VK_PIPELINE_STAGE_2_COLOR_ATTACHMENT_OUTPUT_BIT`
- these are `SP_CS`
  - `VK_PIPELINE_STAGE_2_COMPUTE_SHADER_BIT`
- these are transfers
  - `VK_PIPELINE_STAGE_2_COPY_BIT`
  - `VK_PIPELINE_STAGE_2_RESOLVE_BIT`
  - `VK_PIPELINE_STAGE_2_BLIT_BIT`
  - `VK_PIPELINE_STAGE_2_CLEAR_BIT`
  - when `CP_BLIT`, they use GRAS, `SP_FS`, and RB
  - when falling back to `CP_DRAW`, they use the entire graphipcs pipeline
- these are host
  - `VK_PIPELINE_STAGE_2_HOST_BIT`
- these map to other stages
  - `VK_PIPELINE_STAGE_2_ALL_GRAPHICS_BIT`
  - `VK_PIPELINE_STAGE_2_ALL_TRANSFER_BIT`
  - `VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT`
  - `VK_PIPELINE_STAGE_2_VERTEX_INPUT_BIT`
  - `VK_PIPELINE_STAGE_2_NONE`
  - `VK_PIPELINE_STAGE_2_TOP_OF_PIPE_BIT` is deprecated
    - is `VK_PIPELINE_STAGE_2_NONE` in the first scope
    - is `VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT` with no access flag in the
      second scope
  - `VK_PIPELINE_STAGE_2_BOTTOM_OF_PIPE_BIT` is deprecated
    - is `VK_PIPELINE_STAGE_2_ALL_COMMANDS_BIT` with no access flag in the
      first scope
    - is `VK_PIPELINE_STAGE_2_NONE` in the second scope
- `vk2tu_single_stage`
  - when in the first scope, we can pick the correct stage, the latest of
    multiple correct stages, or any later stage
  - when in the second scope, we can pick the correct stage, the earliest of
    multiple correct stages, or any earlier stage
  - `TU_STAGE_CP` if CP
    - except for `VK_PIPELINE_STAGE_2_INDEX_INPUT_BIT`
  - `TU_STAGE_FE` if VFD
  - `TU_STAGE_SP_VS` if `SP_{V,H,D,G}S`
  - `TU_STAGE_SP_PS` if `SP_FS`
    - also includes compute
  - `TU_STAGE_PS` if VPS, GRAS or RB
  - transfers use
    - `TU_STAGE_PS` if first scope
    - `TU_STAGE_SP_PS` if second scope
  - host uses
    - `TU_STAGE_CP` if first scope
    - `TU_STAGE_PS` if second scope
- `tu_flush_for_stage`
  - because flushes/invalidates happen at end of pipe, when there are pending
    flushes/invalidates, we have to move the first scope to end of pipe
  - `TU_CMD_FLAG_WAIT_FOR_IDLE`
  - `TU_CMD_FLAG_WAIT_FOR_ME`

## `VkAccessFlagBits2`

- these are from CP and are uncached
  - `VK_ACCESS_2_INDEX_READ_BIT`
  - `VK_ACCESS_2_INDIRECT_COMMAND_READ_BIT`
  - `VK_ACCESS_2_TRANSFORM_FEEDBACK_COUNTER_READ_BIT_EXT`
  - `VK_ACCESS_2_TRANSFORM_FEEDBACK_COUNTER_WRITE_BIT_EXT`
  - `VK_ACCESS_2_CONDITIONAL_RENDERING_READ_BIT_EXT`
- these are from VFD and are UCHE cached
  - `VK_ACCESS_2_VERTEX_ATTRIBUTE_READ_BIT`
- these are from VPC and are UCHE cached
  - `VK_ACCESS_2_TRANSFORM_FEEDBACK_WRITE_BIT_EXT`
- these are from SP and are UCHE (or L1) cached
  - `VK_ACCESS_2_UNIFORM_READ_BIT`
  - `VK_ACCESS_2_INPUT_ATTACHMENT_READ_BIT`
  - `VK_ACCESS_2_SHADER_SAMPLED_READ_BIT`
  - `VK_ACCESS_2_SHADER_STORAGE_READ_BIT`
  - `VK_ACCESS_2_SHADER_STORAGE_WRITE_BIT`
  - `VK_ACCESS_2_TRANSFER_READ_BIT`
    - `CP_BLIT` is how we transfer in most cases, and is equivalent to sysmem
      rendering
    - `CP_DRAW` is a fallback, and is also equivalent to sysmem rendering
    - it is thus L1 cached for `TRANSFER_READ`
- these are from RB and can be CCU cached or uncached
  - `VK_ACCESS_2_COLOR_ATTACHMENT_READ_BIT`
  - `VK_ACCESS_2_COLOR_ATTACHMENT_WRITE_BIT`
  - `VK_ACCESS_2_DEPTH_STENCIL_ATTACHMENT_READ_BIT`
  - `VK_ACCESS_2_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT`
  - `VK_ACCESS_2_TRANSFER_WRITE_BIT`
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
  - `VK_ACCESS_2_HOST_READ_BIT`
  - `VK_ACCESS_2_HOST_WRITE_BIT`
  - it can be uncached and thus coherent
  - it can also be CPU cached and not coherent
  - or it can be CPU cached but coherent
    - when GPU snoops or the cache is an LLC
- these map to other access flags
  - `VK_ACCESS_2_MEMORY_READ_BIT`
  - `VK_ACCESS_2_MEMORY_WRITE_BIT`
  - `VK_ACCESS_2_SHADER_READ_BIT`
  - `VK_ACCESS_2_SHADER_WRITE_BIT`
- `vk2tu_access`
  - `TU_ACCESS_UCHE_READ` if UCHE (or L1) cached reads
  - `TU_ACCESS_UCHE_WRITE` if UCHE cached writes
  - `TU_ACCESS_CCU_COLOR_READ` is unused
    - `TU_ACCESS_CCU_COLOR_INCOHERENT_READ` is used instead for CCU color
      cached reads
  - `TU_ACCESS_CCU_COLOR_WRITE` if CCU color cached writes
    - well, only when `VK_ACCESS_2_TRANSFER_WRITE_BIT`
    - `TU_ACCESS_CCU_COLOR_INCOHERENT_WRITE` is used instead for the rest
    - see `tu_cmd_access_mask` for the reason
  - `TU_ACCESS_CCU_DEPTH_READ` is unsued
    - `TU_ACCESS_CCU_DEPTH_INCOHERENT_READ` is used instead for CCU depth
      cached reads
  - `TU_ACCESS_CCU_DEPTH_WRITE` is unused
    - `TU_ACCESS_CCU_DEPTH_INCOHERENT_WRITE` is used instead for CCU depth
      cached writes
  - `TU_ACCESS_SYSMEM_READ` is used for uncached reads
    - note that this includes tile load and host reads
  - `TU_ACCESS_SYSMEM_WRITE` is used for uncached writes
    - note that this includes tile store and host writes
    - except `TU_ACCESS_CP_WRITE` is used instead for
      `VK_ACCESS_2_TRANSFORM_FEEDBACK_COUNTER_WRITE_BIT_EXT`
- `tu_flush_for_access`
  - for the first scope, we only care about writes
    - `pending_flush_bits` are what need to happen for the first scope writes
      to be available to host and visible to device
    - they do not necessarily happen if there is no second scope
    - only `pending_flush_bits` is updated when inspecting the first scope
  - for the second scope, we treat reads and writes the same, because reads
    can have RAW data hazard and writes can have WAW data hazard
    - `flush_bits` are what must happen before the next draw or blit or end of
      command buffer
    - only `flush_bits` is updated when inspecting the second scope
  - once we work out `pending_flush_bits` and `flush_bits`, we can remove the
    duplicated bits from `pending_flush_bits`.  We do not emit commands right
    away because we like to defer until we must emit 
    - this is good because we have to assume the worst (sysmem rendering) when
      outside of the render pass.  We can skip flushes if the second scope
      ends up using gmem rendering?
    - apps might generate unnecessary barriers?

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

## Queries

- `tu_GetQueryPoolResults`
  - `query_slot` has an `available` member that is set by gpu
  - `query_result_addr` expects result(s) to follow `query_slot` immediately
  - if `VK_QUERY_RESULT_WAIT_BIT` and the result is not available,
    `wait_for_available` busy waits
  - otherwise, the results are copied out
- `VK_QUERY_TYPE_OCCLUSION`
  - a `occlusion_query_slot` has
    - `available`
    - `result`
    - `begin`
    - `end`
  - `CP_EVENT_WRITE(ZPASS_DONE)` writes out the count of fragments that pass
    depth test
    - `PERF_CP_ZPASS_DONE` is a counter for fragments that pass depth test
    - `RB_SAMPLE_COUNT_CONTROL` specifies to copy out the counter
    - `RB_SAMPLE_COUNT_ADDR` specifies the dst address
  - `tu_CmdBeginQuery` emits `CP_EVENT_WRITE(ZPASS_DONE)` to update `begin`
  - `tu_CmdEndQuery` updates `end` and `result`
    - it uses `CP_EVENT_WRITE(ZPASS_DONE)` to update `end`
    - to be able to wait for the update, it
      - uses `CP_MEM_WRITE` to init `end` to ~0 first
      - uses `CP_WAIT_REG_MEM` to wait for the update on gpu
      - uses `CP_MEM_TO_MEM` to accumulate `end - begin` to `result`
      - uses `CP_MEM_WRITE` to set `available`
    - accumulation is cruicial because, when in a render pass, these cmds are
      emitted to `draw_cs` and are executed for each tile
- `VK_QUERY_TYPE_PRIMITIVES_GENERATED_EXT`
  - a `primitives_generated_query_slot` has
    - `available`
    - `result`
    - `begin`
    - `end`
  - `tu_CmdBeginQuery` updates `begin`
    - `CP_EVENT_WRITE(START_PRIMITIVE_CTRS)` starts the counter
    - `CP_REG_TO_MEM` to copy `REG_A6XX_RBBM_PRIMCTR_7` to `begin`
    - it carefully does this only in binning or sysmem mode, rather than
      per-tile, with some help from `tu6_render_tile`
  - `tu_CmdEndQuery` updates `end` and `result`
    - `CP_REG_TO_MEM` to copy `REG_A6XX_RBBM_PRIMCTR_7` to `end`
    - `CP_MEM_TO_MEM` to accumulate `end - begin` to `result`
    - `CP_EVENT_WRITE(STOP_PRIMITIVE_CTRS)` to stop the counter

## Vertex Inputs

- `tu_pipeline_builder_parse_vertex_input`
  - it parses `VkPipelineVertexInputStateCreateInfo` and emits
    - `VFD_FETCH_STRIDE(i)`
    - `VFD_DECODE_INSTR(i)`
    - `VFD_DECODE_STEP_RATE(i)`
  - it uses `tu6_emit_vertex_input` as the helper, which can handle the
    dynamic `VkVertexInputBindingDescription2EXT` and
    `VkVertexInputAttributeDescription2EXT`
- `tu6_emit_program`
  - `tu6_emit_vfd_dest` parses vs inputs and emits
    - `VFD_CONTROL_0`
    - `VFD_DEST_CNTL_INSTR`
  - `tu6_emit_vs_system_values` parses vs inputs and emits
    - `VFD_CONTROL_1..6`
- `tu_CmdBindVertexBuffers2EXT` emits
  - `VFD_FETCH_BASE(i)`
  - `VFD_FETCH_SIZE(i)`
- dynamic states
  - `tu_pipeline_builder_parse_dynamic` translates `VK_DYNAMIC_STATE_*` to
    `TU_DYNAMIC_STATE_*`
  - both `tu_pipeline` and `tu_cmd_state` have
    `dynamic_state[TU_DYNAMIC_STATE_COUNT]`
    - bits in `pipeline->dynamic_state_mask` decide which to use
- `VK_DYNAMIC_STATE_VERTEX_INPUT_BINDING_STRIDE`
  - it is translated to `TU_DYNAMIC_STATE_VB_STRIDE`
  - when not set, `VFD_FETCH_STRIDE` is emitted to
    `pipeline->dynamic_state[TU_DYNAMIC_STATE_VB_STRIDE]`
    - `tu_CmdBindPipeline` calls `tu_cs_emit_draw_state` on the stateobj
  - when set, `VFD_FETCH_STRIDE` is deferred until
    `tu_CmdBindVertexBuffers2EXT`
    - `tu6_emit_vertex_strides` emits `VFD_FETCH_STRIDE` to
      `cmd->state.dynamic_state[TU_DYNAMIC_STATE_VB_STRIDE]` and sets
      `TU_CMD_DIRTY_VB_STRIDE`
    - `tu6_draw_common` checks the dirty bit and calls `tu_cs_emit_draw_state`
- `VK_DYNAMIC_STATE_VERTEX_INPUT_EXT`
  - it is translated to `TU_DYNAMIC_STATE_VERTEX_INPUT` and
    `TU_DYNAMIC_STATE_VERTEX_INPUT`
    - it is a super set of `VK_DYNAMIC_STATE_VERTEX_INPUT_BINDING_STRIDE`
- to summarize,
  - `tu_CmdBindVertexBuffers2EXT` emits to `cmd->state.vertex_buffers` and
    sets `TU_CMD_DIRTY_VERTEX_BUFFERS`
    - if `VK_DYNAMIC_STATE_VERTEX_INPUT_BINDING_STRIDE`,
      `tu6_emit_vertex_strides` emits to
      `cmd->state.dynamic_state[TU_DYNAMIC_STATE_VB_STRIDE]` and sets
      `TU_CMD_DIRTY_VB_STRIDE`
  - `tu_CmdBindPipeline`, when not dynamic, emits dynamic states from
    `pipeline->dynamic_state[..]`
    - `tu_update_num_vbs` reduces the stateobj sizes using `pipeline->num_vbs`
  - `tu_CmdSetVertexInputEXT`
  - `tu6_draw_common`
    - emits `TU_DRAW_STATE_VB` if `TU_CMD_DIRTY_VERTEX_BUFFERS`
    - emits `TU_DYNAMIC_STATE_VB_STRIDE` if `TU_CMD_DIRTY_VB_STRIDE` and
      dynamic

## Shaders

- debug environment variables
  - `TU_DEBUG=nir` prints the result of `vk_spirv_to_nir`
    - useful for spirv-to-nir issues
    - it prints `translated nir:` to stderr first for each translation
    - it calls `nir_print_shader(nir, stderr)`
  - `NIR_DEBUG=print` or `NIR_DEBUG=print_vs` to print the result after each
    lowering/optimization
    - useful for nir issues
    - it prints the name of the pass to stdout first
    - it calls `nir_print_shader(nir, stdout)` after each pass
  - `IR3_SHADER_DEBUG=disasm` or `IR3_SHADER_DEBUG=vs` prints finalized nir
    and disassembled binary code
    - if `disasm`,
      - in `ir3_finalize_nir`,
        - it calls `mesa_logi("----------------------");`
        - it calls `nir_log_shaderi` twice, before and after
      - after `ir3_nir_post_finalize`, it calls
        - `mesa_logi("dump nir%d: type=%d", shader->id, shader->type)`
	- it also calls `nir_log_shaderi`
      - in `ir3_nir_lower_variant`,
        - it calls `mesa_logi("----------------------");`
        - it calls `nir_log_shaderi` twice, before and after
    - before assembly, it calls `nir_log_shaderi`
      - `mesa_logi("NIR (final form) for %s shader %s:", ir3_shader_stage(so), so->name);`
    - after assembly, it calls `ir3_shader_disasm`
      - `"Native code%s for unnamed %s shader %s with sha1 %s:\n"`
  - `IR3_SHADER_OVERRIDE_PATH=<dir>`
    - replace shaders by `<dir>/<sha1>.asm`, where sha1 is from
      `IR3_SHADER_DEBUG=disasm`
- `vk_common_CreateShaderModule` is used
  - the common entrypoint just saves and hashes SPIR-V for later use
  - the common `vk_shader_module` has `nir`, which is only used by radv to
    wrap internally-created nir shaders using
    `vk_shader_module_handle_from_nir`
- `vkCreate*Pipelines`, more specifically
  `tu_pipeline_builder_compile_shaders`, is where everything happens
- `tu_pipeline_builder_compile_shaders`
  - it collects `VkPipelineShaderStageCreateInfo` into `stage_infos` ordered
    by stages
  - it initializes a `tu_shader_key` for each stage
    - this is only used by turnip
    - not for variants tracked by `ir3_shader`
  - it initializes a single `ir3_shader_key` for all stages
    - turnip does not make use of `ir3_shader` variants
  - it initializes `pipeline_sha1` for cache
    - inputs are pipeline layout, `ir3_shader_key`, compile options, plus
    - spirv sha1s, names of entrypoints, specializations, and `tu_shader_key`s
  - it calls `tu_spirv_to_nir` to translate spirv to nir for each stage
    - plus common lowerings
  - it calls `tu_link_shaders` to link varyings between stages
    - plus dropping dead varyings and compact varyings
  - it calls `tu_shader_create` to create a `tu_shader` for each stage
    - this applies all lowerings and optimizations on the nir
    - `ir3_finalize_nir` is called to finalize the nir (to be consumed by ir3
      backend compiler)
    - `ir3_shader_from_nir` is called to create `ir3_shader`
  - it calls `tu_shaders_init` to create a `tu_compiled_shaders`
    - `tu_compiled_shaders` can be serialized into or deserialized out of the
      pipeline cache
  - finally, it calls `ir3_shader_create_variant` for each stage
    - this creates `ir3_shader_variant` and compiles finalized nir to the
      binary code
    - internally, `ir3_nir_post_finalize` is called to really finalize nir

## Draw State Groups

- `enum tu_draw_state_group_id` is a list of draw state groups
  - a draw state group is a driver-defined stateobj, a small gpu buffer
    containing gpu commands
  - when a pipeline is created, many draw state groups are allocated and
    initialized
    - the more dynamic states the pipeline has, the less draw state groups are
      allocated
  - there are also draw state groups not covered by a pipeline
- when a pipeline is created, many draw state groups are initialized
  - for a draw state group that has a fixed size, `tu_cs_draw_state` is used
    to sub-allocate the bo
  - for a draw state group that has a dynamic size, `tu_cs_begin_sub_stream`
    is used to sub-allocate the bo
- `CP_SET_DRAW_STATE`
  - outside of a render pass, `tu_disable_draw_states` is used to disable all
    groups
    - the important hw bit is `CP_SET_DRAW_STATE__0_DISABLE_ALL_GROUPS`
    - the function sets `TU_CMD_DIRTY_DRAW_STATE`
  - `tu6_draw_common` re-emits all dirty draw state gropus
  - `tu_CmdBindPipeline` emits many draw state gropus, unless deferred to
    `tu6_draw_common` when `TU_CMD_DIRTY_DRAW_STATE` is set
- pipeline dynamic states
  - `tu_pipeline_builder_parse_dynamic` handles
    `VkPipelineDynamicStateCreateInfo`
    - `pipeline->dynamic_state_mask` is the mask of `TU_DYNAMIC_STATE_blah`
    - `pipeline->blah` is the reg value derived from both static and dynamic
      states
    - `pipeline->blah_mask` is the mask of static states
  - `tu_pipeline_static_state` returns true when a state is static
    - it suballocates a bo (stateobj) decribed by `tu_draw_state`
    - the caller encodes the commands to the bo right away
    - when dyanmic, it returns false
  - take `GRAS_SU_CNTL` register for example
    - when the related dynamic states are set, such as by `tu_CmdSetLineWidth`,
      the values are stored in `cmd->state.gras_su_cntl` and
      `TU_CMD_DIRTY_GRAS_SU_CNTL` is set
    - `tu6_draw_common` emits the register
    - `tu_CmdBindPipeline` has `UPDATE_REG` macro to do the same

## Const Register File

- drivers use `CP_LOAD_STATE6_GEOM/FRAG` to load data into the register file 
  - `CP_LOAD_STATE6_0_DST_OFF` and `CP_LOAD_STATE6_0_NUM_UNIT` are in vec4's
  - type is `ST6_CONSTANTS`, to load into the const register file
  - `SS6_DIRECT` if data is in the pkt payload; `SS6_INDIRECT` if data is at
    iova
  - block is `SB6_VS_SHADER` to `SB6_FS_SHADER`
- `struct ir3_const_state`
  - layout of the const register file
  - `ir3_setup_const_state`
  - `reserved_user_consts`
    - units are vec4's
    - this is a `ir3_shader_options` that drivers can use
    - drivers use `nir_intrinsic_load_uniform` to access
  - `ubo_state::size`
    - units are bytes
    - `ir3_nir_analyze_ubo_ranges` and `ir3_nir_lower_ubo_loads` can lower
      `nir_intrinsic_load_ubo` to `nir_intrinsic_load_uniform`, if the range
      can be statically determined and there is space in the const register
      file
  - `preamble_size`
    - units are vec4's
    - ir3 uses `nir_intrinsic_store_preamble` to init
    - ir3 uses `nir_intrinsic_load_preamble` to access
    - transparent to drivers
  - `num_ubos`
    - for iovas of UBOs
    - isn't turnip bindless?
  - `image_dims.count`
    - only for a5xx-
    - a6xx uses `ir3_RESINFO`
  - `num_driver_params`
    - units are dwords
    - various params (`ir3_driver_param`) to be uploaded by drivers
  - `primitive_param`
  - `primitive_map`
  - `immediates_count`
    - units are dwords
    - some immediates cannot be encoded in the instructions.  `lower_immed`
      lowers them use const register file
- `ir3_shader_variant::constlen`
  - `ir3_collect_info` calls `collect_reg_info` to find the max const register
    accessed and saves it in `max_const`
  - `constlen` is set to `max_const + 1`, thus the units are vec4s
  - when the const register file is relatively addressed, `constlen` is set to
    cover all consts
- `ir3_trim_constlen`
  - `max_const_geom` is 512
  - `max_const_safe` is 128
  - VS to GS share the same const register file, whose size is indicated by
    `max_const_geom`
  - this function marks which stages should be recompiled with `safe_constlen`
    such that their constlen is capped at 128
- push constants
  - `tu_shader_create` calls `nir_lower_explicit_io` with
    `nir_var_mem_push_const` to lower `nir_intrinsic_load_deref` to
    `nir_intrinsic_load_push_constant`
  - `gather_push_constants` scans all `nir_intrinsic_load_push_constant` to
    determine min/max byte offsets
  - `lower_load_push_constant` lowers `nir_intrinsic_load_push_constant` to
    `nir_intrinsic_load_uniform`
    - note that spirv only has UBOs (loaded with `nir_intrinsic_load_ubo`) but
      no uniforms

## IR3 Public APIs

- drivers include `ir3_comiler.h`, `ir3_shader.h`, and `ir3_nir.h` to access
  the public APIs
- `tu_CreateDevice`
  - `ir3_compiler_create` creates the top-level `struct ir3_compiler`
  - turnip sets `disable_cache` and manages disk cache itself
- `tu_DestroyDevice`
  - `ir3_compiler_destroy` destroys the top-level `struct ir3_compiler`
- `tu_pipeline_builder_compile_shaders` or `tu_compute_pipeline_create`
  - `tu_spirv_to_nir` converts spirv to nir
    - `ir3_get_compiler_options` to get `nir_options`
    - `ir3_optimize_loop` applies generic optimizations
  - `tu_shader_create` creates `ir3_shader`
    - `ir3_nir_lower_io_to_temporaries` applies some std passes
    - `ir3_finalize_nir` applies more passes and the nir is ready
    - `ir3_shader_from_nir` creates `ir3_shader` to wrap nir
  - `ir3_shader_create_variant` created `ir3_shader_variant`
    - this is the most heavy function
  - `ir3_trim_constlen` checks variants do not exceed `constlen` limits
    - drivers need to re-create variants with `safe_constlen` if they exceed
      the limits
  - `tu_shader_destroy`
    - `ir3_shader_destroy` destroys `ir3_shader` now that turnip has
      `ir3_shader_variant`
- `tu_pipeline_builder_parse_*`
  - `ir3_const_state` gets the const register file layout
  - `ir3_shader_branchstack_hw`
  - `ir3_link_shaders` updates `ir3_shader_linkage` for vs and fs
  - `ir3_link_stream_out` updates `ir3_shader_linkage` for vs and so
  - `ir3_link_add` updates `ir3_shader_linkage` even more
  - `ir3_find_output_regid` is for linkage of outputs
  - `ir3_find_sysval_regid` is for linkage of sysvals
- misc
  - pipeline cache
    - `ir3_store_variant`
    - `ir3_retrieve_variant`
  - `ir3_shader_get_variant`
    - it is the same as `ir3_shader_create_variant`, except the memory is
      managed by `ir3_shader`

## IR3 Source Code Structure

- driver apis
  - `ir3_compiler.c`
    - `ir3_compiler_create`, top-level function for  use by drivers
    - `ir3_disk_cache.c` is disabled by turnip
  - `ir3_shader.c`
    - `ir3_shader_from_nir`
    - `ir3_shader_create_variant`
    - `ir3_shader_get_variant`
    - `ir3_shader_assemble`
    - `ir3_shader_disasm`
      - disabbembly for debug
    - `ir3_shader_outputs`, a3xx only
    - `ir3_link_stream_out` adds so linkage
- `ir3_shader_create_variant` complies nir
  - `ir3_nir.c` provides nir post-processing
    - `ir3_finalize_nir` has been called by drivers
    - `ir3_nir_post_finalize` is called once before creating variants
      - `ir3_nir_lower_load_barycentric_at_offset.c`
      - `ir3_nir_lower_load_barycentric_at_sample.c`
      - `ir3_nir_move_varying_inputs.c`
      - `ir3_nir_trig.py`
    - `ir3_nir_lower_variant` is called from `ir3_context_init` (which is
      called from `ir3_compile_shader_nir`) for each variant
      - `ir3_nir_lower_tess.c`
      - `ir3_nir_lower_wide_load_store.c`
      - `ir3_nir_lower_64b.c`
      - `ir3_nir_opt_preamble.c`
      - `ir3_nir_analyze_ubo_ranges.c`
      - `ir3_nir_lower_io_offsets.c`
  - `ir3_context.c` manages nir to ir3 translation
    - `ir3_context_init` applies NIR passes before the translation
      - `ir3_nir_lower_variant`
      - `ir3_nir_imul.py`
      - `ir3_nir_lower_tex_prefetch.c`
  - `ir3_compiler_nir.c` translates nir to ir3
    - `ir3_compile_shader_nir`
      - `ir3_context_init` is called to prepare nir for translation
      - `emit_instructions` translates `nir_instr` to `ir3_instruction`
        - `OPC_META_*` and `OPC_*_MACRO` are pseudo instructions that will be
          lowered from various places
      - `ir3.c` is for working with IR3
        - `ir3_a4xx.c` is a4xx backend
        - `ir3_a6xx.c` is a6xx backend
        - `ir3_image.c` is for ibo/ssbo related stuff
	- `ir3_delay.c` is used by `ir3_sched`, `ir3_postsched`, and
	  `ir3_legalize` for nop-delays
      - ir3 passes are applied
        - `ir3_remove_unreachable.c`
        - `ir3_array_to_ssa.c`
        - `ir3_cf.c`
        - `ir3_cp.c`
        - `ir3_cse.c`
        - `ir3_dce.c`
      - `ir3_sched.c` schedules instructions (to minimize nop-delays) pre-RA
      - `ir3_ra.c` performs register allocation and spilling
        - `ir3_dominance.c`
        - `ir3_liveness.c`
        - `ir3_merge_regs.c`
        - `ir3_spill.c`
        - `ir3_lower_spill.c`
        - `ir3_lower_parallelcopy.c`
      - `ir3_postsched.c` schedules instructions post-RA
      - `ir3_lower_subgroups.c` lowers  `OPC_MACRO_*` that are
      	subgroup-related
      - `ir3_legalize.c`
        - it also lowers `OPC_DSXPP_MACRO`
      - `ir3_legalize.c`
      - `collect_tex_prefetches` lowers `OPC_META_TEX_PREFETCH`
  - `assemble_variant` generates the native code
    - `ir3_shader_assemble` calls `isa_assemble` from `libir3encode`
- utils
  - `ir3_print.c` provides debug prints
  - `ir3_validate.c` and `ir3_ra_validate.c` are only enabled on debug build
    for validations
  - `disasm-a3xx.c` and `ir3_assembler.c` are only used by tools

## IR3 VS IOs

- spirv
  - `OpVariable`
  - `OpLoad`
  - `OpStore`
- nir
  - `decl_var`
  - `deref_var`
  - `load_deref`
  - `store_deref`
- `nir_lower_io`
  - `load_input`
  - `store_output`
- `ir3_compile_shader_nir`
  - `setup_input` lowers `nir_intrinsic_load_input` to `OPC_META_INPUT`
    - `ir3_legalize` will discard most meta instructions including
    	`OPC_META_INPUT`
  - `create_sysval_input`
  - `setup_output` lowers `nir_intrinsic_store_output` to nothing
- `ir3_shader_variant`
  - `outputs_count` and `outputs`
    - these are `gl_varying_slot`
  - `output_size` is used only when HS/DS/GS is enabled
  - `inputs_count` and `inputs`
    - these include `gl_vert_attrib` and `gl_system_value`
  - `input_size` is 0 because it is only for HS/DS/GS
  - `total_in`, `sysval_in`, and `varying_in` are set but are not used for
    VS
    - they are mostly used for fs before a6xx
  - `tu6_emit_vertex_input` uses `inputs` to emit `A6XX_VFD_DEST_CNTL_INSTR`
- when HS/DS/GS is enabled
  - vs uses `chsh` to jump to the next stage
  - `ir3_nir_lower_to_explicit_output` initializes `output_loc` and
    `output_size`
    - it uses `shader_io_get_unique_index` to map `gl_varying_slot` to byte
    	offsets
    - `output_size` is in dwords (e.g., 2 out's have 8 dwords)
    - `store_output` is lowered to `store_shared_ir3`
  - system values
    - `SYSTEM_VALUE_TCS_HEADER_IR3` is written to
    	`VARYING_SLOT_TCS_HEADER_IR3`
    - `SYSTEM_VALUE_REL_PATCH_ID_IR3` is written to
    	`VARYING_SLOT_REL_PATCH_ID_IR3`
    - `SYSTEM_VALUE_GS_HEADER_IR3` is written to
    	`VARYING_SLOT_GS_HEADER_IR3`
    - `SYSTEM_VALUE_PRIMITIVE_ID` is written to `VARYING_SLOT_PRIMITIVE_ID`

## IR3 HS/DS IOs

- spirv
  - `OpVariable`
  - `OpLoad`
  - `OpStore`
  - `AccessChain`
- nir
  - `decl_var`
  - `deref_var`
  - `load_deref`
  - `store_deref`
  - `deref_array`
- `nir_lower_io`
  - `load_per_vertex_input`
  - `store_per_vertex_output`
- ir3-specific tess lowering
  - `ir3_nir_lower_tess_ctrl`
    - `load_per_vertex_output` and `load_output` are lowered to
      `load_global_ir3`
    - `store_per_vertex_output` and `store_output` are lowered to
      `store_global_ir3`
    - these use a bo of size `TU_TESS_BO_SIZE`
  - `ir3_nir_lower_tess_eval`
    - `load_per_vertex_input` and `load_input` are lowered to
      `load_global_ir3`
  - `ir3_nir_lower_to_explicit_output`
    - `store_output` is lowered to `store_shared_ir3` and uses the shared
      (local) memory
    - `output_loc` and `output_size` describe the layout of the local memory
  - `ir3_nir_lower_to_explicit_input`
    - `load_per_vertex_input` is lowered to `load_shared_ir3`
  - VS
    - when followed by HS/DS/GS, use `ir3_nir_lower_to_explicit_output` to
      lower outputs to share (local) memory
    - otherwise, no special handling
  - HS is always followed by DS
    - use `ir3_nir_lower_tess_ctrl` to lower outputs to a `TU_TESS_BO_SIZE` bo
    - use `ir3_nir_lower_to_explicit_input` to lower inputs to share (local)
      memory
  - DS is always preceded by HS
    - use `ir3_nir_lower_tess_eval` to lower inputs to a `TU_TESS_BO_SIZE` bo
    - if followed by GS, also use `ir3_nir_lower_to_explicit_output` to lower
      outputs to share (local) memory
  - GS is always preceded by VS/HS/DS
    - use `ir3_nir_lower_to_explicit_input`
- HS `ir3_shader_variant`
  - `inputs_count` and `inputs`
    - `gl_varying_slot` has been lowered to local memory
    - these are system values such as `SYSTEM_VALUE_TCS_HEADER_IR3` and
      `SYSTEM_VALUE_REL_PATCH_ID_IR3`
  - `input_size` is set by `ir3_nir_lower_to_explicit_input` and is the max
    unique index plus 1
  - `outputs_count` and `outputs` are 0
  - `output_size` is set by `ir3_nir_lower_tess_ctrl`
- DS `ir3_shader_variant`
  - `inputs_count` and `inputs` are the same as in HS
  - `input_size` is by `ir3_nir_lower_tess_eval` and is the max unique index
    plus 1
  - with GS, `outputs_count` and `outputs` are 0.  `output_size` is set.
  - without GS, `outputs_count` and `outputs` are for fs.  `output_size` is 0.

## IR3 UBOs

- descriptors
  - `descriptor_size` returns `A6XX_TEX_CONST_DWORDS * 4`, 64 bytes, for UBOs
  - `write_ubo_descriptor` uses only 8 bytes
  - I guess this is because adreno bindless expects descriptors to be
    multiples of 64 bytes
- spirv-to-nir
  - use `nir_intrinsic_vulkan_resource_index` and
    `nir_intrinsic_load_vulkan_descriptor` to load descriptors
- `tu_shader_create`
  - `nir_lower_explicit_io` lowers `nir_intrinsic_load_deref` to
    `nir_intrinsic_load_ubo`
    - the deref, which is a `nir_deref_type_struct`, is also lowered to an
      offset by `nir_explicit_io_address_from_deref`
  - `nir_intrinsic_vulkan_resource_index` is lowered to vec3
    - x is `set`, a constant,
    - y is `offset` to the descriptor set iova (units are 64-bytes)
    - z is shift
  - `nir_intrinsic_load_vulkan_descriptor` passes the vec3 along and forces z
    to 0, because the address format is
    `nir_address_format_vec2_index_32bit_offset`
  - `lower_ssbo_ubo_intrinsic` replaces `(set, offset)` by
    `nir_intrinsic_bindless_resource_ir3(set, offset)`
- `ir3_nir_lower_variant`
  - `nir_intrinsic_bindless_resource_ir3` is untouched
  - `nir_intrinsic_load_ubo` has several outcomes
    - `ir3_nir_lower_ubo_loads` can potentially lower it to
      `nir_intrinsic_load_uniform`
      - `ir3_nir_lower_preamble` can further lower it to
      	`nir_intrinsic_copy_ubo_to_uniform_ir3`
    - otherwise, `nir_lower_ubo_vec4` lowers it to
      `nir_intrinsic_load_ubo_vec4`
- `emit_instructions`
  - `nir_intrinsic_bindless_resource_ir3` is translated to `mov`
    - remember that its src is an offset (in units of 64-bytes)
  - `nir_intrinsic_load_ubo_vec4` is translated to `ldc`
    - because it loads from a bindless resource, `ir3_handle_bindless_cat6`
      sets `IR3_INSTR_B` and sets `base` to `nir_intrinsic_desc_set`
  - `nir_intrinsic_load_uniform` is translated to `mov` from the const reg
    file
  - `nir_intrinsic_copy_ubo_to_uniform_ir3` is translated to `ldc.k`
- bind descriptor set
  - update `REG_A6XX_SP_BINDLESS_BASE` to sets' iovas for all 5 sets
  - update `REG_A6XX_HLSQ_BINDLESS_BASE` to sets' iovas for all 5 sets
  - `A6XX_HLSQ_INVALIDATE_CMD` `A6XX_HLSQ_INVALIDATE_CMD_GFX_BINDLESS`
