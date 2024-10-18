Mesa PanVK Compiler
===================

## Environment Variables

- `PANVK_DEBUG`
  - `nir` dumps lowered nir before passing it to the backend compiler
  - `trace` implies `sync` and dumps all submits
- `BIFROST_MESA_DEBUG` is for the bifrost compiler
  - `msgs` Print debug messages
  - `shaders` Dump shaders in NIR and MIR
  - `shaderdb` Print statistics
  - `verbose` Disassemble verbosely
  - `internal` Dump even internal shaders
  - `nosched` Force trivial bundling
  - `nopsched` Disable scheduling for pressure
  - `inorder` Force in-order bundling
  - `novalidate` Skip IR validation
  - `noopt` Skip optimization passes
  - `noidvs` Disable IDVS
  - `nosb` Disable scoreboarding
  - `nopreload` Disable message preloading
  - `spill` Test register spilling

## `vk_device_shader_ops` and `vk_shader_ops`

- `panvk_per_arch(device_shader_ops)`
  - `panvk_get_nir_options` is called for spirv-to-nir translation or for nir
    deserialization
    - `vk_common_CreateShadersEXT` calls it
    - `vk_common_CreateGraphicsPipelines` and
      `vk_common_CreateComputePipelines` call it too
      - to translate on cache miss or to deserialize on cache hit
  - `panvk_get_spirv_options` is called for spirv-to-nir translation
    - `vk_common_CreateShadersEXT`, `vk_common_CreateGraphicsPipelines`, and
      `vk_common_CreateComputePipelines` call it
  - `panvk_preprocess_nir` is called to pre-process nir right after
    spirv-to-nir translation
    - it is called after translation and before serialization (for cache)
  - `panvk_hash_graphics_state` hashes `vk_graphics_pipeline_state` for cache
  - `panvk_compile_shaders` compiles nir to `vk_shader`
  - `panvk_deserialize_shader` deserializes blob to `vk_shader`
  - `panvk_cmd_bind_shaders` updates `cmd` shader states
    - `vk_common_CmdBindPipeline` calls `vk_graphics_pipeline_cmd_bind` which
      calls it
  - `vk_cmd_set_dynamic_graphics_state` updates `cmd->dynamic_grahpics_states`
    from a pipeline
    - `vk_common_CmdBindPipeline` calls `vk_graphics_pipeline_cmd_bind` which
      calls it
- `panvk_shader_ops`
  - `panvk_shader_destroy` destroys `vk_shader`
  - `panvk_shader_serialize` serializes `vk_shader` to blob
  - `panvk_shader_get_executable_properties`
    - `vk_common_GetPipelineExecutablePropertiesKHR` calls
      `vk_{graphics,compute}_pipeline_get_executable_properties` which calls
      it
  - `panvk_shader_get_executable_statistics`
    - `vk_common_GetPipelineExecutableStatisticsKHR` calls
      `vk_{graphics,compute}_pipeline_get_executable_statistics` which calls it
  - `panvk_shader_get_executable_internal_representations`
    - `vk_common_GetPipelineExecutableInternalRepresentationsKHR` calls
      `vk_{graphics,compute}_pipeline_get_internal_representations` which calls it
- `panvk_get_nir_options` returns `bifrost_nir_options_v9`
  - it is defined by `DEFINE_OPTIONS` macro
- `panvk_get_spirv_options`
  - `ubo_addr_format` and `ssbo_addr_format` are both
    `nir_address_format_vec2_index_32bit_offset`
  - `phys_ssbo_addr_format` is `nir_address_format_64bit_global`
- `panvk_preprocess_nir`
  - `vk_spirv_to_nir` applies generic lowering already
  - if fs, `nir_lower_io_to_vector`
  - `nir_lower_io_to_temporaries`
  - `nir_lower_indirect_derefs`
  - `nir_opt_copy_prop_vars`
  - `nir_opt_combine_stores`
  - `nir_opt_loop`
  - if fs, `nir_lower_input_attachments`
  - `nir_lower_tex`
  - `nir_lower_system_values`
  - `nir_lower_compute_system_values`
  - if fs, `nir_lower_wpos_center`
  - `nir_split_var_copies`
  - `nir_lower_var_copies`
- `panvk_compile_shaders`
  - `panvk_lower_nir`
    - `panvk_per_arch(nir_lower_descriptors)`
    - `nir_split_var_copies`
    - `nir_lower_var_copies`
    - `nir_lower_explicit_io`
    - `valhall_lower_get_ssbo_size`
    - `valhall_pack_buf_idx`
    - if cs, `nir_lower_vars_to_explicit_types` and `nir_lower_explicit_io`
    - `nir_assign_io_var_locations`
    - `nir_shader_gather_info`
    - `pan_shader_preprocess`
      - `bifrost_preprocess_nir`
    - if vs, `pan_lower_image_index`
    - `panvk_lower_sysvals`
  - `panvk_compile_nir`
    - `GENX(pan_shader_compile)`
      - `bifrost_compile_shader_nir`
  - `panvk_shader_upload`
    - the binary is uploaded to `dev->mempools.exec`
    - the shader program descriptor (spd) is uploaded to `dev->mempools.rw`
    - if vs, there are actually 3 binaries and 3 spds: `pos_points`,
      `pos_triangles`, and `var`

## Bifrost Compiler

- `bifrost_preprocess_nir`
  - `nir_lower_vars_to_ssa`
  - if vs,
    - `nir_lower_viewport_transform`
    - `nir_lower_point_size`
  - `nir_lower_global_vars_to_local`
  - `nir_lower_vars_to_scratch`
  - `nir_lower_indirect_derefs`
  - `nir_split_var_copies`
  - `nir_lower_var_copies`
  - `nir_lower_vars_to_ssa`
  - `nir_lower_io`
  - `nir_opt_constant_folding`
  - `nir_lower_mediump_io`
  - if vs,
    - `pan_nir_lower_store_component`
  - if fs,
    - `bi_lower_sample_mask_writes`
    - `bifrost_nir_lower_load_output`
  - `nir_lower_mem_access_bit_sizes`
  - `bi_lower_load_push_const_with_dyn_offset`
  - `nir_lower_ssbo`
  - `pan_lower_sample_pos`
  - `nir_lower_bit_size`
  - `nir_lower_64bit_phis`
  - `pan_lower_helper_invocation`
  - `nir_lower_int64`
  - `nir_opt_idiv_const`
  - `nir_lower_idiv`
  - `nir_lower_tex`
  - `nir_lower_image_atomics_to_global`
  - `nir_lower_alu_to_scalar`
  - `nir_lower_load_const_to_scalar`
  - `nir_lower_phis_to_scalar`
  - `nir_lower_flrp`
  - `nir_lower_var_copies`
  - `nir_lower_alu`
  - `nir_lower_frag_coord_to_pixel_coord`
- `bifrost_compile_shader_nir`
  - `pan_nir_lower_zs_store`
  - `bi_optimize_nir`
  - `bi_compile_variant`
    - `emit_cf_list` converts nir to bir
    - `bi_lower_swizzle`
    - optimize
      - `bi_opt_copy_prop`
      - `bi_opt_constant_fold`
      - `bi_opt_mod_prop_forward`
      - `bi_opt_mod_prop_backward`
      - `bi_opt_dce`
      - `bi_opt_cse`
    - `bi_lower_opt_instructions`
    - `va_optimize`
    - `va_lower_isel`
    - `va_lower_constants`
    - `va_repair_fau`
    - `bi_lower_branch`
    - if fs, `bi_analyze_helper_requirements`
    - optimize
      - `bi_opt_fuse_dual_texture`
      - `bi_opt_cse`
      - `bi_opt_dce`
    - `bi_pressure_schedule`
    - `bi_register_allocate`
    - `bi_opt_post_ra`
    - `va_assign_slots`
    - `va_insert_flow_control_nops`
    - `va_merge_flow`
    - `va_mark_last`
    - `bi_pack_valhall`
