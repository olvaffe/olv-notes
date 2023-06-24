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
