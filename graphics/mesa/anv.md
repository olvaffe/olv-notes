Mesa ANV
========

## Pipelines

- `anv_CreateGraphicsPipelines` calls `anv_graphics_pipeline_create`
- `anv_graphics_pipeline_compile` compiles shaders
  - `anv_graphics_pipeline_load_nir` translates spirv to nir
  - `anv_pipeline_nir_preprocess` preprocesses nir
  - a series of lowering and optimizations
  - `anv_pipeline_compile_vs` and `anv_pipeline_compile_fs` (and others)
    generates the binary
    - they call `brw_compile_vs` and `brw_compile_fs` for codegen, which call
      `brw_postprocess_nir`

## `shaderFloat64`

- newer intel does not support fp64
- some games use fp64 without checking, and `fp64_workaround_enabled` can be
  enabled as a per-game workaround
  - when enabled and when the device does not support fp64, `fp64_nir` is
    created from `src/compiler/glsl/float64.glsl`
  - the glsl file implements various `__fXXX64` functions
  - `nir_lower_doubles_options` has `nir_lower_fp64_full_software`
  - `brw_preprocess_nir` calls `nir_lower_doubles` with `fp64_nir`
- `nir_lower_doubles`
  - `lower_doubles_instr_to_soft` maps `nir_op_XXX` to calls to `__fXXX64` in
    the GLSL

## `shaderInt64`

- newer intel does not support 64-bit integers
  - but anv always advertises `shaderInt64`
  - `brw_compiler_create` enables everything for `nir_lower_int64_options`
  - `brw_preprocess_nir` calls `nir_lower_int64_float_conversions`
  - `brw_postprocess_nir` calls `nir_lower_int64`

## blorp

- `vkCmdCopyBuffer2` flow
  - `blorp_batch_init` initializes `blorp_batch`
  - `blorp_buffer_copy` does the copying
    - `isl_surf_init` to initialize a `isl_surf` on stack for both src/dst
    - `blorp_copy`
      - `blorp_params_init` initializes `blorp_params`
      - `brw_blorp_surface_info_init` initializes `params.src` and
        `params.dst`
      - `blorp_copy_get_formats` overrides formats
      - `blorp_surf_convert_to_uncompressed` overrides w/h for compressed
        formats
      - `genX(blorp_exec)` executes
        - if `BLORP_BATCH_USE_COMPUTE`, `blorp_exec_on_compute` and
          `blorp_exec_compute`
        - else, `blorp_exec_on_render` and `blorp_exec_3d`
  - `blorp_batch_finish` is nop

## Descriptor Sets

- for each descriptor type, `anv_descriptor_data_for_type` decides what does
  the descriptor consist of
  - `ANV_DESCRIPTOR_INDIRECT_SAMPLED_IMAGE` is `anv_sampled_image_descriptor`
  - `ANV_DESCRIPTOR_INDIRECT_STORAGE_IMAGE` is `anv_storage_image_descriptor`
  - `ANV_DESCRIPTOR_INDIRECT_ADDRESS_RANGE` is `anv_address_range_descriptor`
  - because `indirect_descriptors` is usually true, the hw descriptors
    (`RENDER_SURFACE_STATE` and `SAMPLER_STATE`) are not directly written to
    the descriptor sets
