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
  - `tu_CmdClearAttachments` can use `RB_BLIT` or 3D draw depending on whether
    we are in gmem or sysmem mode
  - `tu_clear_sysmem_attachment` uses `r2d_ops` to clear attachments, unless
    msaa
  - `tu_clear_gmem_attachment` uses `RB_BLIT` to clear GMEM on tile load
  - `tu_load_gmem_attachment` uses `RB_BLIT` to load GMEM on tile load
  - `tu_store_gmem_attachment` can use `RB_BLIT`, `r2d_ops`, or `r3d_ops` to
    blit GMEM to attachment on tile store
    - `RB_BLIT` is preferred but requires 16x4 alignment
    - `r2d_ops` is used unless msaa
      - `SP_PS_2D_SRC` needs iova of the source; apparently gmem is mapped and
      	has iova as well
    - `r3d_ops` appears to have bugs..
