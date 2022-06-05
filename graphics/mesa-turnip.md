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

## Shaders

- debug environment variables
  - `TU_DEBUG=nir` prints the result of `vk_spirv_to_nir`
    - useful for spirv-to-nir issues
  - `NIR_DEBUG=print` or `NIR_DEBUG=print_vs` to print the result after each
    lowering/optimization
    - useful for nir issues
  - `IR3_SHADER_DEBUG=disasm` prints finalized nir and disassembled binary
    code
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
